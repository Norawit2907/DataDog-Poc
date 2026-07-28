# Nginx / Service Proxy

## Description

This service is a reverse proxy that routes requests to the frontend, backend, ads, discounts, and DBM services. When viewing information about the application in Datadog, this service is referenced as `service-proxy`.

The final nginx config is generated at container start by [docker-entrypoint.sh](./docker-entrypoint.sh). It runs `envsubst` against [default.conf.template](./default.conf.template), filling in placeholders based on environment variables, then starts nginx with the resulting config.

## A/B Testing Ads services (Optional)

The service-proxy can split traffic between two Ads services based on a percentage. For more information, see the [README.md](../../README.md#ab-testing-ads-services) file in the root of this repo.

Add these environment variables to the `service-proxy` service:

```yaml
environment:
  - ADS_A_UPSTREAM=${ADS_A_UPSTREAM:-ads:3030}
  - ADS_B_UPSTREAM=${ADS_B_UPSTREAM:-ads-python:3030}
  - ADS_B_PERCENT=${ADS_B_PERCENT:-0}
```

## SSL/TLS (Optional)

The service-proxy can terminate its own TLS on port `443`, gated by the `ENABLE_SSL` environment variable. This is needed when a lab exposes Storedog through Instruqt's external ingress (`*.instruqt.io`) rather than the learner proxy (`*.env.play.instruqt.com`), since external ingress does not include Instruqt-provided HTTPS termination. For more information, see the [README.md](../../README.md#enable-ssltls-on-the-service-proxy) file in the root of this repo.

> [!IMPORTANT]
> When `ENABLE_SSL=true`, a certificate and key must be mounted into the container at `/etc/nginx/certs/cert.pem` and `/etc/nginx/certs/key.pem`. Otherwise, the service-proxy exits with an error on startup.

### Using SSL/TLS

#### Docker Compose

1. Add these to the `service-proxy` service in your `docker-compose.yml` file:

    ```yaml
    service-proxy:
      ports:
        - "443:443"
      volumes:
        - ./certs:/etc/nginx/certs:ro
      environment:
        - ENABLE_SSL=${ENABLE_SSL:-true}
    ```

1. Place your certificate and key at `./certs/cert.pem` and `./certs/key.pem`.

1. Start the app via `docker compose up -d`

#### Kubernetes

Replace your entire `k8s-manifests/storedog-app/deployments/nginx.yaml` file with one of the manifests in [k8s-manifests/](./k8s-manifests/) in this directory, depending on how the certificate is delivered to the pod:

- [`nginx-ssl-secret.yaml`](./k8s-manifests/nginx-ssl-secret.yaml) - certificate and key come from a Kubernetes Secret (recommended, portable to any cluster).
- [`nginx-ssl-hostpath.yaml`](./k8s-manifests/nginx-ssl-hostpath.yaml) - certificate and key come from a `hostPath` volume on the node (simpler when the pod always schedules onto the VM that fetched the certificate).

**Option A - Secret**

```bash
kubectl create secret tls storedog-tls \
  --cert=/path/to/cert.pem \
  --key=/path/to/key.pem \
  -n storedog
```

```yaml
volumes:
  - name: apmsocketpath
    hostPath:
      path: /var/run/datadog/
  - name: sslcerts
    secret:
      secretName: storedog-tls
      items:
        - key: tls.crt
          path: cert.pem
        - key: tls.key
          path: key.pem
...
volumeMounts:
  - name: apmsocketpath
    mountPath: /var/run/datadog
  - name: sslcerts
    mountPath: /etc/nginx/certs
    readOnly: true
```

`kubectl create secret tls` always stores the pair as `tls.crt`/`tls.key`; the `items` remap above gives the mounted files the `cert.pem`/`key.pem` names the service-proxy expects.

**Note for Instruqt Kubernetes labs**: in a split control-plane/worker topology, the SSL certificate is typically only fetchable via GCP instance metadata on whichever VM has `provision_ssl_certificate: true` in `config.yml` (usually the VM exposed via external ingress), while `kubectl` admin access usually only exists on the VM that ran `kubeadm init`. Getting the certificate from one VM to the other uses Instruqt's standard host-to-host file transfer pattern: store an SSH keypair as Instruqt `secrets:` entries, install it via each host's `track_scripts/setup-<hostname>` script, then `scp` the certificate over. See [docs.instruqt.com/reference/platform/networking](https://docs.instruqt.com/reference/platform/networking.md) and the official [`instruqt/track-rsync-intro`](https://github.com/instruqt/track-rsync-intro) example track for more on this pattern.

The following are illustrative examples only, showing what this looks like in a lab's `track_scripts/`. They are not part of this repo; a lab adopting `ENABLE_SSL` for Kubernetes would add snippets like these to its own track.

`track_scripts/setup-control-plane` additions:

```bash
# -----------------------------------------------------------------------------
# OPTIONAL: Accept the SSL cert/key from the worker host over SSH
# -----------------------------------------------------------------------------
echo "Installing worker's SSH public key for the SSL cert handoff"
mkdir -p /root/.ssh /tmp/storedog-ssl
echo "${WORKER_SSH_PUBLIC_KEY}" >> /root/.ssh/authorized_keys
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys

# ... existing kubeadm/Datadog/namespace setup happens here ...

# -----------------------------------------------------------------------------
# OPTIONAL: Create the storedog-tls Secret once the worker delivers the cert
# -----------------------------------------------------------------------------
echo "Waiting for the SSL certificate to arrive from the worker host..."
wait_for 30 5 \
    "[ -f /tmp/storedog-ssl/cert.pem ] && [ -f /tmp/storedog-ssl/key.pem ]" \
    "SSL certificate received from worker!" \
    "SSL certificate not received yet" \
    "SSL certificate transfer" || true

if [ -f /tmp/storedog-ssl/cert.pem ] && [ -f /tmp/storedog-ssl/key.pem ]; then
    echo "Creating storedog-tls secret"
    kubectl create secret tls storedog-tls \
        --cert=/tmp/storedog-ssl/cert.pem \
        --key=/tmp/storedog-ssl/key.pem \
        -n storedog
    rm -rf /tmp/storedog-ssl
else
    echo "Warning: SSL certificate never arrived. service-proxy will fail to start with ENABLE_SSL=true."
fi
```

New `track_scripts/setup-worker`:

```bash
#!/bin/bash
set -eo pipefail

echo "Setting up environment for ${INSTRUQT_TRACK_SLUG} (worker)"

echo "Waiting for the Instruqt host bootstrap to finish"
until [ -f /opt/instruqt/bootstrap/host-bootstrap-completed ]
do
  sleep 0.5
done

# -----------------------------------------------------------------------------
# OPTIONAL: Fetch the Instruqt-provisioned SSL cert and hand it to control-plane
# Only this VM has provision_ssl_certificate: true, so only it can read the
# cert from its own GCP instance metadata. control-plane is the only host
# with kubectl admin access, so the cert is copied there via SSH.
# -----------------------------------------------------------------------------
echo "Fetching SSL certificate from GCP instance metadata"
MD="http://metadata.google.internal/computeMetadata/v1/instance/attributes"
mkdir -p /root/certs
curl -sf -H "Metadata-Flavor: Google" "$MD/ssl-certificate" -o /root/certs/cert.pem
curl -sf -H "Metadata-Flavor: Google" "$MD/ssl-certificate-key" -o /root/certs/key.pem

echo "Installing SSH private key for the SSL cert handoff to control-plane"
mkdir -p /root/.ssh
echo "${WORKER_SSH_PRIVATE_KEY}" > /root/.ssh/id_rsa
chmod 700 /root/.ssh
chmod 600 /root/.ssh/id_rsa

echo "Waiting for control-plane to accept SSH connections"
until ssh -o StrictHostKeyChecking=no -o BatchMode=yes -o ConnectTimeout=5 \
    root@control-plane true 2>/dev/null
do
  sleep 5
done

echo "Copying the SSL certificate to control-plane"
scp -o StrictHostKeyChecking=no /root/certs/cert.pem /root/certs/key.pem \
  root@control-plane:/tmp/storedog-ssl/
```

`WORKER_SSH_PUBLIC_KEY` and `WORKER_SSH_PRIVATE_KEY` would be added as `secrets:` entries in that lab's `config.yml`, generated once with `ssh-keygen -t ed25519 -N "" -f worker-key` and stored in the Instruqt secrets store, never checked into the repo.

**Option B - hostPath**

```yaml
volumes:
  - name: apmsocketpath
    hostPath:
      path: /var/run/datadog/
  - name: sslcerts
    hostPath:
      path: /etc/storedog/certs
      type: Directory
...
volumeMounts:
  - name: apmsocketpath
    mountPath: /var/run/datadog
  - name: sslcerts
    mountPath: /etc/nginx/certs
    readOnly: true
```

This avoids the cross-VM credential hop, since the fetch and the pod both happen on the same node, but it is not portable to a cluster where the pod isn't guaranteed to schedule onto that node. A lab using this option needs its own `track_scripts/setup-worker`-style step to fetch the Instruqt-provisioned cert/key into `/etc/storedog/certs` before `kubectl apply`, the same way the `setup-worker` example above pulls it from GCP instance metadata.
