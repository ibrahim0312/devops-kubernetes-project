# DevOps Kubernetes Project

[![Build and Test](https://github.com/ibrahim0312/devops-kubernetes-project/actions/workflows/ci.yaml/badge.svg)](https://github.com/ibrahim0312/devops-kubernetes-project/actions/workflows/ci.yaml)

A hands-on DevOps project by **Shaik Tameem Ibrahim**: build an Nginx website image with Docker, deploy it to a local Kubernetes cluster, and automatically build and test the application with GitHub Actions.

## Project status

- Website and `/health` endpoint created and tested in Docker.
- Single-node kind cluster created and verified healthy.
- Application deployed using a Kubernetes Deployment and ClusterIP Service.
- Readiness and liveness probes, CPU requests, and memory limits configured.
- Source code committed and pushed to GitHub.
- GitHub Actions Docker build and application checks passed.
- Version 2.0 built, loaded into kind, and deployed; CI passed again.

Deployment to the local cluster is manual. The current workflow performs CI; it does not publish images or deploy to Kubernetes. Rollback commands below are an optional exercise, not a recorded completed check.

## Tools and environment

| Component | Purpose |
| --- | --- |
| Windows Command Prompt | Local commands |
| Docker Desktop with Linux containers / WSL2 | Build images and run containers |
| kind | Run a local Kubernetes node in Docker |
| kubectl | Manage Kubernetes resources |
| Nginx (`nginx:stable-alpine`) | Serve the website and health file |
| Git and GitHub | Version control and source hosting |
| GitHub Actions | Automated image build and HTTP checks |

The development laptop has 8 GB RAM. Docker reported 4 CPUs and approximately 3.7 GiB available memory during setup. One Kubernetes node and one application replica keep the lab small. No cloud infrastructure is provisioned.

Cluster name: `devops-labs`  
Kubectl context: `kind-devops-labs`

## Repository files

| Path | Contents |
| --- | --- |
| `app/index.html` | Website displaying the application version |
| `app/health` | Static health response: `healthy` |
| `Dockerfile` | Nginx base image and application files |
| `k8/web.yaml` | Deployment and Service definitions |
| `.github/workflows/ci.yaml` | Build and Test workflow |
| `.gitignore` | Excludes environment files, kubeconfig files, logs, and editor settings |

## Architecture

The browser connects to `localhost:8081` through a running `kubectl port-forward` session targeting the Service. Kubernetes selects an application pod, where Nginx serves the website on port 80.

The standalone Docker test uses `localhost:8080`. The Kubernetes test uses `localhost:8081`. These are separate application instances.

## Run locally

The following command blocks are for **Windows Command Prompt**. Keep Docker Desktop running in Linux container mode. Paste YAML and HTML into an editor, not into the terminal.

### 1. Verify prerequisites

```cmd
docker info
git --version
kubectl version --client
kind version
```

During this build, winget could not locate kind. We installed the official Windows AMD64 executable instead:

```cmd
if not exist "%USERPROFILE%\tools" mkdir "%USERPROFILE%\tools"
curl.exe --fail --location "https://kind.sigs.k8s.io/dl/v0.33.0/kind-windows-amd64" --output "%USERPROFILE%\tools\kind.exe"
set "PATH=%USERPROFILE%\tools;%PATH%"
kind version
```

This PATH change lasts only for the current Command Prompt window. The download above records the version used for this project, rather than promising the latest release.

### 2. Clone the repository

For a fresh checkout:

```cmd
cd /d "%USERPROFILE%"
git clone https://github.com/ibrahim0312/devops-kubernetes-project.git
cd devops-kubernetes-project
```

If the project already exists locally, use its existing folder instead of cloning again.

### 3. Create the cluster

Run this only if `devops-labs` does not already exist:

```cmd
kind get clusters
kind create cluster --name devops-labs --wait 5m
```

Verify:

```cmd
kubectl --context kind-devops-labs get nodes
kubectl --context kind-devops-labs get pods -n kube-system
```

The node should be `Ready`, and system pods should be running.

### 4. Build and test the current application

The current manifest uses `devops-web:v2`:

```cmd
docker build -t devops-web:v2 .
docker run -d --name devops-web-local -p 8080:80 devops-web:v2
curl.exe http://localhost:8080/health
```

Expected health response: `healthy`. Open http://localhost:8080 to view the page.

If the earlier standalone container still exists, remove only that test container before running its replacement:

```cmd
docker rm -f devops-web-local
```

Then repeat the `docker run` command. This does not remove the kind cluster.

### 5. Load and deploy the image

```cmd
kind load docker-image devops-web:v2 --name devops-labs
kubectl --context kind-devops-labs apply -f k8\web.yaml
kubectl --context kind-devops-labs rollout status deployment/devops-web --timeout=120s
kubectl --context kind-devops-labs get pods
kubectl --context kind-devops-labs get service devops-web
```

The application pod should show `1/1 Running`.

### 6. Access the Kubernetes application

```cmd
kubectl --context kind-devops-labs port-forward service/devops-web 8081:80
```

Leave this terminal open. Visit http://localhost:8081 and check the endpoint from a second terminal:

```cmd
curl.exe http://localhost:8081/health
```

Press Ctrl+C in the forwarding terminal to stop forwarding. This does not stop the deployment.

## Kubernetes configuration

| Setting | Value / behavior |
| --- | --- |
| Deployment and Service | `devops-web` |
| Replicas | 1 |
| Image | `devops-web:v2` |
| Image pull policy | `Never`; load the matching image into kind first |
| Container / Service port | 80 |
| Service type | ClusterIP; access locally using port-forward |
| CPU request / limit | `50m` / `250m` |
| Memory request / limit | `32Mi` / `128Mi` |
| Readiness probe | HTTP `/health`, initial delay 3 seconds, period 5 seconds |
| Liveness probe | HTTP `/health`, initial delay 10 seconds, period 10 seconds |

Readiness controls whether a pod can receive Service traffic. Repeated liveness failures cause Kubernetes to restart the container. The static health file checks that Nginx can serve a request; it does not check external dependencies.

## Rolling update performed

The initial deployment used `devops-web:v1` and displayed Version 1.0. We changed the HTML badge to Version 2.0, built `devops-web:v2`, loaded it into kind, updated the image in `k8/web.yaml`, and applied the manifest.

For a future version, use a new image tag, load that image, update the manifest to match, and apply it:

```cmd
kubectl --context kind-devops-labs apply -f k8\web.yaml
kubectl --context kind-devops-labs rollout status deployment/devops-web --timeout=120s
```

Restart port-forward after a rollout if its selected pod was replaced. Commit both the website and manifest changes so the repository records the intended deployment.

## Optional rollback exercise

These commands require a previous revision to exist. A fresh deployment may have no earlier revision to restore.

```cmd
kubectl --context kind-devops-labs rollout history deployment/devops-web
kubectl --context kind-devops-labs rollout undo deployment/devops-web
kubectl --context kind-devops-labs rollout status deployment/devops-web --timeout=120s
kubectl --context kind-devops-labs get deployment devops-web -o=jsonpath="{.spec.template.spec.containers[0].image}"
```

An undo changes the live deployment but does not edit the YAML. Restore the version declared in the repository with:

```cmd
kubectl --context kind-devops-labs apply -f k8\web.yaml
kubectl --context kind-devops-labs rollout status deployment/devops-web --timeout=120s
```

## Continuous integration

Workflow: `.github/workflows/ci.yaml`  
Name: **Build and Test**

It runs on pushes and pull requests targeting `main`, and can also be started manually. On an Ubuntu runner it:

1. Checks out the source using `actions/checkout@v4`.
2. Builds the image as `devops-web:ci`.
3. Starts a test container on port 8080.
4. Waits for `/health` and verifies the response is `healthy`.
5. Checks the website contains `Tameem's DevOps Lab`.
6. Prints container logs and removes the test container, including after failures.

The job has a 10-minute timeout and read-only repository contents permission. No registry login or Kubernetes credentials are needed for these checks.

View runs: https://github.com/ibrahim0312/devops-kubernetes-project/actions

## Troubleshooting encountered during the build

| Problem | Cause and resolution |
| --- | --- |
| `docker --info` fails | Use `docker info`. |
| `kubectl --client` fails | Use `kubectl version --client`. |
| kind is not recognized | Install its executable and add its folder to PATH. |
| Docker cannot find Dockerfile | Save the file as exactly `Dockerfile`, not `Dockerfile.txt`, in the build folder. |
| Image pull denied after a failed build | Build successfully before running the image. Docker login was not needed for the local application image. |
| YAML lines report “not recognized” | YAML belongs in `k8\web.yaml` in an editor; only commands belong in Command Prompt. |
| kubectl disappears after pasting YAML | `path: /health` changed the terminal PATH. Close that window and open a fresh Command Prompt. |
| `type k8/web.yaml` fails | Use Windows path syntax: `type k8\web.yaml`. |
| Manifest path cannot be found | The actual project folder is `k8`, not `k8s`. |
| Wrong cluster context | Use `kind-devops-labs`, including the final `s`. |
| `ErrImageNeverPull` | Load the exact image tag from the manifest into the correct kind cluster. |
| Page stops opening after an update | Restart the port-forward session. |

For deployment problems, inspect:

```cmd
kubectl --context kind-devops-labs get pods
kubectl --context kind-devops-labs describe pods -l app=devops-web
kubectl --context kind-devops-labs logs deployment/devops-web
```

## Stop working and resume

Stop port-forward with Ctrl+C. Save and commit source changes. Access requires Docker Desktop, the cluster, and a new port-forward session when you return. Avoid deleting the cluster if you want to preserve its live resources.

## Future improvements

- Automated Kubernetes deployment testing in a temporary CI cluster.
- Image publishing and a separate deployment workflow.
- Security scanning and pinned image/action versions.
- Additional application features and monitoring.

These improvements are planned; they are not part of the completed implementation.
