# kube_samples

## Deploy the Sample Service

There are two variants of the sample web service:

- `simple-web-service.yaml` overwrites `index.html` inside the main container before nginx starts.
- `simple-web-service-init.yaml` uses an init container to write `index.html` into a shared `emptyDir` volume.

Deploy one or both, depending on what you want to demonstrate.

### Direct container rewrite (`simple-web-service.yaml`)

Apply and verify:

- `kubectl apply -f simple-web-service.yaml`
- `kubectl rollout restart deployment simple-web-service`
- `kubectl rollout status deployment simple-web-service`
- `kubectl delete deployment simple-web-service && kubectl apply -f simple-web-service.yaml`

View logs to confirm the startup script ran:

- `kubectl logs -f deployment/simple-web-service`

Remove everything when done:

- `kubectl delete -f simple-web-service.yaml`

#### YAML walkthrough

- **Deployment**
  - `apiVersion`, `kind`, `metadata`: identify this as a Deployment named `simple-web-service`.
  - `spec.replicas`: runs two identical pods for redundancy.
  - `spec.selector` + `spec.template.metadata.labels`: tie the Deployment to the pods it manages (`app: simple-web-service`).
  - `spec.template.spec.containers[0]`: defines the nginx container.
    - `image`: pulls `nginx:stable`.
    - `env`: exposes the pod name (`POD_NAME`) so the startup script can read it.
    - `command` / `args`: replace the default entrypoint, write a custom `index.html`, then start nginx.
    - `ports`: documents the container’s HTTP port (80) for probes and services.
    - `readinessProbe` / `livenessProbe`: periodic HTTP checks so Kubernetes knows when the pod can receive traffic and when it needs a restart.
- **Service**
  - `spec.type: LoadBalancer`: requests an external IP when supported; also exposes a NodePort.
  - `spec.selector`: forwards traffic to pods with `app: simple-web-service`.
  - `spec.ports`: maps external port 443 and nodePort 30443 to the container’s HTTP port.

### Init container rewrite (`simple-web-service-init.yaml`)

Apply and verify:

- `kubectl apply -f simple-web-service-init.yaml`
- `kubectl rollout status deployment simple-web-service-init`

Key differences:

- **Init container** (`init-html`): runs once before the main container starts, writing `index.html` with the pod name. This guarantees the file exists before nginx launches, and isolates the bootstrap logic outside the runtime container.
- **Shared volume** (`emptyDir` + `volumeMounts`): the init container writes the file into an `emptyDir` volume mounted at `/workdir`. The nginx container mounts the same volume at `/usr/share/nginx/html`, so it serves the file created during initialization.

#### YAML walkthrough

- **Deployment**
  - `spec.template.spec.initContainers[0]`: short-lived `busybox` container that writes `index.html` into `/workdir`.
    - `env`: pulls the pod name into `POD_NAME`.
    - `command` / `args`: shell script that logs its actions and writes the file.
    - `volumeMounts`: attaches the shared `html` volume at `/workdir`.
  - `spec.template.spec.containers[0]`: the main nginx container.
    - `volumeMounts`: mounts the same `html` volume at `/usr/share/nginx/html`, so nginx serves the file written by the init container.
    - Probes and port definition match the direct-rewrite deployment.
  - `spec.template.spec.volumes[0]`: `emptyDir` volume provisioned on the node; exists only as long as the pod runs and is empty at pod start.
- **Service**
  - Mirrors the direct version but uses `app: simple-web-service-init` labels.
  - Exposes HTTPS/NodePort traffic on port 31443 while forwarding to the pod’s port 80.

View init container logs:

- `kubectl logs deployment/simple-web-service-init -c init-html --previous`
- `kubectl logs -f deployment/simple-web-service-init`

Remove everything when done:

- `kubectl delete -f simple-web-service-init.yaml`

## Demonstrate Load Balancing

Each request must use a fresh TCP connection to see both pods:

- `curl -H "Connection: close" http://192.168.49.2:30443/index.html`
- `curl --http1.0 http://192.168.49.2:30443/index.html`

Use DevTools in Chrome and disable cache, or open new incognito windows, to achieve the same effect.

For the init-container service, replace the host: `http://192.168.49.2:31443/index.html`.