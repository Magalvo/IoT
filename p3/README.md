# Part 3 - K3d and Argo CD

This folder implements Part 3 with:

- one local K3d cluster named `iot`;
- an `argocd` namespace for Argo CD;
- a `dev` namespace managed by Argo CD;
- the public `wil42/playground:v1` image;
- automatic GitOps synchronization from a public GitHub repository; and
- the application exposed at `http://localhost:8888`.

Run everything inside a Linux VM, as required by the subject. A practical lab
size is 2 vCPUs, 4 GB RAM, and 20 GB free disk space.

## K3s vs K3d

K3s is the lightweight Kubernetes distribution running the cluster. K3d is a
wrapper that runs K3s server and agent nodes as Docker containers, which makes
local clusters quick to create and remove.

## 1. Prepare the public GitHub repository

The subject requires the repository name to contain a team member's login. Push
this project to a public GitHub repository before running the cluster setup.

```bash
git init
git add .
git commit -m "Set up IoT Part 3"
git branch -M main
git remote add origin https://github.com/YOUR_LOGIN/YOUR_LOGIN-iot.git
git push -u origin main
```

The manifests Argo CD watches are in `p3/confs/dev`.

## 2. Install the tools in the Linux VM

The installer supports current Ubuntu and Debian releases. Run it as the normal
VM user, not as root:

```bash
bash p3/scripts/install.sh
```

If Docker was newly installed, refresh group membership before continuing:

```bash
newgrp docker
```

The script installs Docker Engine, K3d v5.9.0, and a Kubernetes 1.32-compatible
`kubectl` release.

## 3. Create the cluster and deploy Argo CD

Pass the HTTPS URL of the public GitHub repository:

```bash
bash p3/scripts/setup.sh https://github.com/YOUR_LOGIN/YOUR_LOGIN-iot.git
```

Optional arguments are the Git revision and manifest path:

```bash
bash p3/scripts/setup.sh REPOSITORY_URL main p3/confs/dev
```

The setup installs Argo CD v3.4.2, creates both required namespaces, registers
the `iot-app` Application, and waits for the first synchronization.

## 4. Verify the result

```bash
bash p3/scripts/verify.sh
curl http://localhost:8888/
```

The response should contain `"message": "v1"`.

To open the Argo CD UI, run the following command and keep it open:

```bash
bash p3/scripts/argocd-ui.sh
```

Then visit `https://localhost:8080`, accept the local self-signed certificate,
and sign in as `admin` with the password printed by the script.

## 5. Demonstrate the v1 to v2 GitOps update

Edit only the image tag in `p3/confs/dev/deployment.yaml`:

```yaml
image: wil42/playground:v2
```

Commit and push the change:

```bash
git add p3/confs/dev/deployment.yaml
git commit -m "Deploy playground v2"
git push
```

Argo CD polls Git automatically. Watch the rollout and confirm the new response:

```bash
kubectl get application iot-app -n argocd -w
kubectl rollout status deployment/playground -n dev --timeout=5m
curl http://localhost:8888/
```

The response should now contain `"message": "v2"`. Normal Argo CD polling can
take a few minutes. For a quicker defense demo, request an immediate refresh
without manually syncing the application:

```bash
kubectl annotate application iot-app -n argocd \
  argocd.argoproj.io/refresh=hard --overwrite
```

## Useful commands

```bash
kubectl get namespaces
kubectl get pods -n argocd
kubectl get pods -n dev
kubectl get application iot-app -n argocd
k3d cluster list
```

Remove the lab cluster with:

```bash
bash p3/scripts/cleanup.sh
```

## Local Windows test with Docker Desktop

This path is useful for development only; use the Linux VM workflow above for
the evaluation. Start Docker Desktop in Linux-container mode, push the project
to the public GitHub repository, and run PowerShell from the repository root:

```powershell
powershell -ExecutionPolicy Bypass -File p3/scripts/windows/setup.ps1 `
  -RepoUrl https://github.com/YOUR_LOGIN/YOUR_LOGIN-iot.git

powershell -ExecutionPolicy Bypass -File p3/scripts/windows/verify.ps1
```

The setup downloads a checksum-verified K3d executable into the ignored
`p3/.tools` directory and uses the `kubectl` bundled with Docker Desktop.

Open the Argo CD UI with:

```powershell
powershell -ExecutionPolicy Bypass -File p3/scripts/windows/argocd-ui.ps1
```

Remove the Windows test cluster with:

```powershell
powershell -ExecutionPolicy Bypass -File p3/scripts/windows/cleanup.ps1
```
