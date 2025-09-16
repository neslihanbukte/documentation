## 1. Install and configure a K3s cluster

#### Prerequisites

- 3 Ubuntu Server VMs:
  - `k3s-master` (Master) → `10.34.19.64`
  - `k3s-worker1` (Worker 1) → `10.34.19.65`
  - `k3s-worker2` (Worker 2) → `10.34.19.66`

Update all packages:

```bash
sudo apt update && sudo apt upgrade -y
```

#### Turn off swap

```bash
sudo swapoff -a 
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

#### Set hostname (must be different for each node)

```bash
sudo hostnamectl set-hostname k3s-master   # for master
```
```bash
sudo hostnamectl set-hostname k3s-worker1  # for worker 1
```
```bash
sudo hostnamectl set-hostname k3s-worker2  # for worker 2
```

#### Add static host entries (same configuration on all nodes)

```bash
sudo tee -a /etc/hosts > /dev/null <<EOF
10.34.19.64 k3s-master
10.34.19.65 k3s-worker1
10.34.19.66 k3s-worker2
EOF
```

#### Install K3s on the master node 

```bash
curl -sfL https://get.k3s.io | sh -
```

After installation, k3s starts automatically. Verify: 

```bash
sudo kubectl get nodes
```

#### Get the master node token:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```
Save this output - you'll need it when joining worker nodes.

Also note the master's IP address (e.g., `192.168.1.10`).

#### Worker Node Setup

Execute this command on each worker node:

```bash
curl -sfL https://get.k3s.io | K3S_URL="https://<MASTER_NODE_IP>:6443" K3S_TOKEN="<NODE_TOKEN>" sh -
```

<MASTER_NODE_IP>: Master node's IP address

<NODE_TOKEN>: Token from /var/lib/rancher/k3s/server/node-token on the master node

#### Cluster Verification

On the master node:

```bash
sudo kubectl get nodes -o wide
```

#### Using kubectl

By default, `kubectl` only works with root. To use it with a regular user:

```bash
mkdir -p $HOME/.kube
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $USER:$USER $HOME/.kube/config
```
Now you can:

```bash
kubectl get pods -A
```

## 2. Install and configure a GitLab CI runner/agent
- Connect it to your GitLab project

#### Add GitLab Repository

```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
```

#### Install GitLab Runner
```bash
sudo apt install gitlab-runner -y
```

#### Verify Installation

```bash
gitlab-runner --version
```

#### Create a project runner in gitlab
1. Go to your GitLab project.
2. Navigate to: Settings → CI/CD → Runners.
3. Click "Create project runner".
4. Provide: 
<ul>
  <li><strong>Description:</strong> e.g., my-project-runner</li>
  <li><strong>Tags:</strong> optional, e.g., docker,linux</li>
</ul>

After clicking Create runner, GitLab will generate:
<ul>
  <li>a registration token</li>
  <li>the exact registration command to run on your machine</li>
</ul>


#### Register the runner

```bash
sudo gitlab-runner register
```

You will be prompted for:

<ul>
  <li>GitLab instance URL: https://gitlab.com/ (or your GitLab URL)</li>
  <li>Registration token: (your registration token)</li>
  <li>Runner description: k3s (or your preferred name)</li>
  <li>Runner executor: docker</li>
</ul>


#### Start and enable gitlab runner

```bash
# enable auto-start on boot
sudo systemctl enable gitlab-runner
```

```bash
# start gitLab runner service
sudo systemctl start gitlab-runner
```

```bash
# verify runner status
sudo gitlab-runner status
```

```bash
# list registered runners
sudo gitlab-runner list
``` 

## 3. Package the application as a Docker image
- add a dockerfile under code folder 

We will containerize the application using Docker. The project structure includes a `code/` folder where we will add the `Dockerfile`.

Create a file named `Dockerfile` inside the `code/` directory with the following content:

#### Create Dockerfile

```bash
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

#### Build the docker image

```bash
cd code
```

```bash
# build the docker image
docker build -t my-python-app:latest .
```


## 4. Use GitLab project registry to upload Docker images
- create an access token to read & write private registry


We will push our application image to the **GitLab Container Registry** so that it can be used in the CI/CD pipeline.

---

#### Check Container Registry

1. Go to your GitLab project
2. Navigate to Deploy → Container Registry
3. Note your registry URL: registry.gitlab.com/neslihanbukte-group/neslihanbukte-project

#### Create Deploy Token

1. Go to your GitLab project
2. Navigate to Settings → Repository
3. Expand Deploy tokens section
4. Click Add token
5. Fill in the token details:

<ul>
  <li>Name: docker-registry-token</li>
  <li>Expiration date: (optional)</li>
  <li>Select scopes:
    <ul>
      <li>✅ read_registry</li>
      <li>✅ write_registry</li>
    </ul>
  </li>
</ul>

6. Click Create deploy token
7. Copy and save both username and token (you won't see them again)


## 5. Create a GitLab CI/CD pipeline
- Use `gitlab-ci.yml` to build and deploy the Python application into the Kubernetes cluster using Helm



#### helm chart structure 

```bash
devops-challenge/
├── code/
│ ├── app.py
│ ├── Dockerfile
│ └── requirements.txt
├── chart/
│ ├── Chart.yaml
│ ├── values.yaml
│ └── templates/
│      ├── deployment.yaml
│      ├── service.yaml
│      └── _helpers.tpl
├── infra/
│ ├── gitlab-runner/
│ └── postgres/
├── .gitlab-ci.yml
└── README.md
```
