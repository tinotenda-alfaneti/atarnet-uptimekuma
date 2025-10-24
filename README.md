# atarnet-uptimekuma
uptime-kuma for tracking my homelab apps health

Helm chart and Jenkins pipeline for running [Uptime Kuma](https://github.com/louislam/uptime-kuma) in a microk8s-based homelab cluster.

## Repository Layout
- `charts/app` – Helm chart responsible for deploying the application and its Kubernetes objects.
- `ci/kubernetes/trivy.yaml` – Kubernetes Job manifest the pipeline uses to run container image scans with Trivy.
- `Jenkinsfile` – Declarative pipeline that installs tooling, runs Trivy, deploys the chart, and executes Helm tests.

## Prerequisites
- Kubernetes cluster (tested with microk8s) and `kubectl` access.
- Helm 3.x.
- An ingress controller such as nginx for the included `Ingress` resource.
- A `StorageClass` named `microk8s-hostpath` or an updated value set for `storageClassName`.

## Quick Start (manual deployment)
1. Clone the repository and change into its directory.
2. Review the defaults in `charts/app/values.yaml` and override as needed (e.g., host name, storage size).
3. Deploy or upgrade the release:
   ```bash
   helm upgrade --install uptime-kuma charts/app \
     --namespace monitoring \
     --create-namespace \
     -f charts/app/values.yaml
   ```
4. Inspect the rollout and the bound PVC:
   ```bash
   kubectl get all -n monitoring
   kubectl get pvc -n monitoring
   ```
5. Run the chart hook tests to verify basic connectivity:
   ```bash
   helm test uptime-kuma --namespace monitoring --logs
   ```

## Configuration
All tunable values live in `charts/app/values.yaml`. Key options include:

| Key | Purpose | Default |
| --- | --- | --- |
| `namespace` | Target namespace for all resources | `monitoring` |
| `name` | Base name used for Kubernetes objects | `uptime-kuma` |
| `host` | Hostname exposed by the ingress | `uptime.atarnet.org` |
| `image` | Container image reference | `louislam/uptime-kuma:latest` |
| `storageClassName` | StorageClass for the PVC | `microk8s-hostpath` |
| `storage` | Persistent volume size | `1Gi` |

Override values with `--set key=value` or a custom YAML file when running `helm upgrade --install`.

## Jenkins Pipeline
The declarative pipeline in `Jenkinsfile` automates deployment with the following stages:
1. **Checkout Code** – Pulls the repo into the Jenkins workspace and prepares a `bin/` directory for tooling.
2. **Install Tools** – Downloads architecture-specific `kubectl` and Helm 3 binaries into the workspace.
3. **Verify Cluster Access** – Copies the provided kubeconfig secret and confirms connectivity with `kubectl cluster-info`.
4. **Prepare Namespace & Secrets** – Ensures the target namespace exists prior to deploying the chart.
5. **Trivy Scan** – Creates the Kubernetes `Job` defined in `ci/kubernetes/trivy.yaml` to scan the target image for HIGH/CRITICAL vulnerabilities.
6. **Deploy with Helm** – Performs `helm upgrade --install` using the repository chart, waiting for the deployment to complete.
7. **Verify Deployment** – Executes `helm test` to run the bundled connection test pod and stream logs.

### Required Jenkins Configuration
- Pipeline credentials named `kubeconfigglobal` that supply a kubeconfig file (`KUBECONFIG_CRED`).
- Build agents must allow outbound HTTPS to download Helm/kubectl and reach the Kubernetes API endpoint defined by the kubeconfig.

## Trivy Scan Job
The Kubernetes job in `ci/kubernetes/trivy.yaml` is templated at runtime by the pipeline to target `IMAGE_NAME:TAG`. Adjust the manifest if you need additional flags (e.g., `--exit-code 1` or `--timeout`). The pipeline removes the job during the `post` section to keep the namespace tidy.

## Development Tips
- Validate template rendering locally with `helm lint charts/app` and `helm template charts/app` before pushing changes.
- When modifying pipeline behaviour, test stages that run `kubectl` or `helm` with the same kubeconfig the pipeline uses to avoid environment drift.
- Keep ingress hostnames and storage requirements in sync with your homelab inventory so the chart remains deployable without edits to multiple files.
