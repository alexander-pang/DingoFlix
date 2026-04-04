# Build and Run K3s on macOS with Multipass and k3d

This guide shows how to build K3s from source inside a Linux VM using Multipass, then export the built Docker image and run it locally on macOS with k3d.

> Note: K3s builds require a Linux environment. On macOS, the recommended path is to use Multipass to build inside Ubuntu, then use k3d on the host.

## Prerequisites

- macOS with Homebrew installed
- Docker Desktop installed and running
- Multipass installed
- k3d installed
- A stable internet connection for downloads

## 1. Install required tools on macOS

```bash
brew install multipass
brew install k3d
brew install kubectl
brew install docker
```

Verify the tools:

```bash
multipass version
k3d version
kubectl version --client
docker version
```

## 2. Launch the Multipass build VM

Create a Linux VM for building K3s:

```bash
multipass launch --name k3sServer --cpus 2 --memory 3G --disk 20G
multipass shell k3sServer
```

Inside the VM, you should see a prompt like:

```bash
ubuntu@k3sServer:~$
```

## 3. Prepare the build environment inside Multipass

Install Docker and build tools:

```bash
sudo apt update
sudo apt install -y docker.io git make gcc
sudo usermod -aG docker $USER
newgrp docker
```

If Docker has DNS or networking issues, you can add a simple daemon config:

```bash
cat <<'EOF' | sudo tee /etc/docker/daemon.json
{
  "dns": ["8.8.8.8", "1.1.1.1"]
}
EOF
sudo systemctl restart docker
```

## 4. Clone and build K3s

Clone the official repository and build:

```bash
git clone --depth 1 https://github.com/k3s-io/k3s.git
cd k3s
make download
make generate
SKIP_VALIDATE=true make
```

> If the build is successful, Docker images for K3s are generated locally inside the VM.

## 5. Export the built K3s image to your Mac

Find the built image name, then save it:

```bash
sudo docker images | grep rancher/k3s
sudo docker save rancher/k3s:<tag> | gzip > /tmp/k3s-built.tar.gz
```

Exit the VM and transfer the image to macOS:

```bash
exit
multipass transfer k3sServer:/tmp/k3s-built.tar.gz ~/Downloads/
```

Load the image into Docker on macOS:

```bash
docker load -i ~/Downloads/k3s-built.tar.gz
```

## 6. Create a k3d cluster with the custom image

Create a cluster using the imported image:

```bash
k3d cluster create my-k3s --image rancher/k3s:<tag> --wait
```

Confirm the cluster is running:

```bash
k3d cluster list
kubectl get nodes
kubectl get pods -A
```

## 7. Verify the cluster

Verify that core system pods are running:

```bash
kubectl get pods -n kube-system
```

If the cluster is healthy, you should see `coredns`, `metrics-server`, `traefik`, and `local-path-provisioner` pods in `Running` or `Completed` state.

## 8. Cleanup

To remove the k3d cluster:

```bash
k3d cluster delete my-k3s
```

To remove the Multipass VM:

```bash
multipass delete k3sServer
multipass purge
```

## Notes

- This is a Mac host workflow: the build happens in Linux, while runtime happens in Docker on macOS.
- k3d uses a containerized K3s server inside Docker, so macOS kernel limitations do not block the cluster.
- If you only need a local K3s cluster for development, you can also use `k3d cluster create` with a standard published image, skipping the build step.
