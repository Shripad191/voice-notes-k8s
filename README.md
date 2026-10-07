# Voice Notes on Kubernetes — Minikube on AWS EC2 (Docker + nginx + Kubernetes)

A browser-based voice-to-text notes app, containerized with Docker, served by nginx, and deployed as a replicated workload on a **Kubernetes cluster running in Minikube on a single AWS EC2 instance**.

Open the app, click **Start Dictation**, and your speech is transcribed live into an editable notes area that can be downloaded as a `.txt` file. The application is static; the focus of this project is running it the way real services run on Kubernetes: replicated Pods, a stable Service, health checks, rolling updates, rollbacks, and self-healing.

---

## Architecture

```
   Laptop (Chrome)
        │  http://localhost:8080
        │  SSH tunnel  (ssh -L 8080:localhost:8080 ubuntu@<EC2-IP>)
        ▼
 ┌─────────────────────────── AWS EC2 (Ubuntu 24.04, t3.medium) ───────────────────────────┐
 │                                                                                          │
 │   kubectl port-forward svc/voice-notes 8080:80                                           │
 │        │                                                                                 │
 │   ┌────▼────────────── Minikube node (Docker container, 192.168.49.2) ───────────────┐   │
 │   │                                                                                   │   │
 │   │   Service: voice-notes  ──selector app=voice-notes──▶  Pod 1: nginx + index.html  │   │
 │   │   (NodePort 30090 → :80)                          └──▶  Pod 2: nginx + index.html  │   │
 │   │                                      managed by Deployment: voice-notes (2 replicas) │
 │   └───────────────────────────────────────────────────────────────────────────────────┘   │
 └──────────────────────────────────────────────────────────────────────────────────────────┘
```

| Component | Role |
|---|---|
| **AWS EC2** | Ubuntu 24.04 host (`t3.medium`, 2 vCPU / 4 GB, 30 GB gp3) that runs Docker and Minikube |
| **Minikube (Docker driver)** | Single-node Kubernetes cluster running as a Docker container on the instance |
| **Docker image `voice-notes:v1`** | `nginx:1.27-alpine` serving the app's `index.html` on port 80 |
| **Deployment `voice-notes`** | Keeps 2 replicas running, performs rolling updates, keeps revision history for rollback |
| **Service `voice-notes`** | Stable virtual IP + DNS name that load-balances across Ready Pods via the `app=voice-notes` label |
| **Readiness probe** | `GET /` on port 80; a Pod receives traffic only after nginx is actually serving |
| **SSH tunnel + `kubectl port-forward`** | Delivers the app to the laptop as `http://localhost:8080`, a secure context the browser's microphone API requires |

**Proof this architecture is real, not just a diagram:**

**Screenshot:** `docs/screenshots/01-ec2-instance.png`
*The EC2 instance in the AWS console: state Running, type `t3.medium`, status checks passed.*

**Screenshot:** `docs/screenshots/02-ec2-details.png`
*Instance details: Ubuntu 24.04 AMI and the 30 GB root volume.*

**Screenshot:** `docs/screenshots/03-ec2-security-group.png`
*Inbound rules: only SSH (22) open, restricted to a single source IP. No application port is exposed to the internet.*

**Screenshot:** `docs/screenshots/08-minikube-container.png`
*`docker ps` on the instance: the entire Kubernetes node is one Docker container named `minikube`.*

**Screenshot:** `docs/screenshots/12-k8s-resources.png`
*`kubectl get deploy,pods,svc -o wide`: the Deployment at 2/2, two Running Pods, and the NodePort Service.*

---

## Project Structure

```
voice-notes-k8s/
├── index.html            # The Voice Notes web app (HTML + Tailwind + Web Speech API)
├── Dockerfile            # nginx:1.27-alpine image serving index.html
├── deploy.yaml           # Kubernetes Deployment (2 replicas) + NodePort Service
├── docs/
│   └── screenshots/      # Proof-of-concept screenshots referenced in this README
├── .gitignore
└── README.md
```

---

## Prerequisites

- An AWS account and an EC2 instance running **Ubuntu 24.04**, at least **`t3.medium`** (Minikube needs 2 vCPUs and 2 GB of free memory)
- A security group allowing **SSH (22)** from your IP only
- On the instance: **Docker**, **kubectl** and **Minikube**, run as the `ubuntu` user (the Docker driver refuses to run as root)
- On your laptop: an SSH client, your `.pem` key, and **Chrome or Edge** (the Web Speech API is not available in Firefox)

**Screenshot:** `docs/screenshots/04-ec2-ssh-specs.png`
*SSH session on the instance: OS, CPU count, memory and disk.*

**Screenshot:** `docs/screenshots/05-tool-versions.png`
*Installed versions of Docker, kubectl and Minikube.*

---

## Set Up the Cluster

```bash
minikube start --driver=docker --cpus=2 --memory=3000mb
minikube addons enable metrics-server

minikube status
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

**Screenshot:** `docs/screenshots/06-minikube-start.png`

**Screenshot:** `docs/screenshots/07-minikube-status.png`
*Host, kubelet and API server Running; the single node is `Ready`.*

**Screenshot:** `docs/screenshots/09-minikube-system-pods.png`
*Control-plane and system Pods (etcd, kube-apiserver, scheduler, CoreDNS, kube-proxy) all Running.*

**Screenshot:** `docs/screenshots/10-minikube-addons.png`
*Enabled addons and `kubectl top nodes` output from metrics-server.*

---

## Deploy

```bash
git clone https://github.com/<your-username>/voice-notes-k8s.git
cd voice-notes-k8s

# Build the image and copy it into Minikube's own image store
docker build -t voice-notes:v1 .
minikube image load voice-notes:v1

# Validate, then apply
kubectl apply -f deploy.yaml --dry-run=server
kubectl apply -f deploy.yaml
kubectl rollout status deploy/voice-notes
```

This builds the nginx image, loads it into the Minikube node, and creates a Deployment with two replicas plus a NodePort Service in front of them.

**Screenshot:** `docs/screenshots/11-image-build-load.png`
*`docker build` output and `minikube image ls` confirming `voice-notes:v1` is inside the cluster.*

---

## Test

**Confirm the Service is wired to the Pods:**
```bash
kubectl get endpointslices -l kubernetes.io/service-name=voice-notes
curl -s $(minikube service voice-notes --url) | grep "<title>"
```
The EndpointSlice lists two Pod IPs, and `curl` returns `<title>Voice Notes</title>`.

**Screenshot:** `docs/screenshots/13-endpoints-curl.png`

**Open the app from your laptop.** On the laptop:
```bash
ssh -i <your-key>.pem -L 8080:localhost:8080 ubuntu@<EC2-PUBLIC-IP>
```
Inside that SSH session:
```bash
kubectl port-forward svc/voice-notes 8080:80
```
Then browse to `http://localhost:8080`.

**Screenshot:** `docs/screenshots/14-port-forward.png`
*The SSH tunnel and `Forwarding from 127.0.0.1:8080 -> 80`.*

**Screenshot:** `docs/screenshots/15-app-ui.png`

**Use the app end to end.** Click **Start Dictation**, allow microphone access, speak, then click **Download**.

**Screenshot:** `docs/screenshots/16-dictation-working.png`
*Status "Listening..." with live transcribed text in the notes area.*

**Screenshot:** `docs/screenshots/17-download.png`
*The downloaded `voice_notes.txt` containing the transcribed text.*

**Verify resilience** (self-healing, scaling, rollout, rollback):
```bash
kubectl delete pod -l app=voice-notes --wait=false
kubectl get pods -w                         # replacements are created automatically

kubectl scale deploy/voice-notes --replicas=4
kubectl rollout history deploy/voice-notes
kubectl rollout undo deploy/voice-notes
```

**Screenshot:** `docs/screenshots/18-self-healing.png`
*Deleted Pods are immediately replaced by the Deployment's ReplicaSet with no manual action.*

---

## Update to a New Version

```bash
docker build -t voice-notes:v2 .
minikube image load voice-notes:v2
kubectl set image deploy/voice-notes web=voice-notes:v2
kubectl rollout status deploy/voice-notes
```
Restart the `port-forward` after a rollout, since it stays attached to the Pod it originally connected to.

---

## Tear Down

```bash
kubectl delete -f deploy.yaml
minikube image rm voice-notes:v1
minikube delete
```
Then **stop** the EC2 instance when idle, or **terminate** it and delete any leftover EBS volumes when you're done with the project.

---

## Cost Notes

| Resource | Notes |
|---|---|
| **EC2 `t3.medium`** | Not Free Tier eligible (the Free Tier micro instances are too small for Minikube). Billed per hour only while **running** |
| **EBS gp3 (30 GB)** | Billed while the volume exists, including when the instance is stopped |
| **Data transfer** | Negligible; traffic goes through the SSH session |

Stopping the instance between sessions is the single biggest saving. Minikube and all Kubernetes objects survive a stop/start; just run `minikube start` again.

---

## Design Decisions

- **nginx on Alpine to serve a static app.** The app is a single HTML file, so a small, well-known web server image keeps the container lightweight and fast to roll out.
- **Deployment with 2 replicas, not a bare Pod.** Gives self-healing, rolling updates and rollback history, and demonstrates load-balancing across Pods via the Service.
- **Labels and selectors as the only link between objects.** The Service finds Pods by `app=voice-notes`; the Deployment's `matchLabels`, the Pod template labels and the Service selector are kept identical.
- **Readiness probe on `/`.** New or restarted Pods only receive traffic once nginx is serving, which keeps rolling updates zero-downtime.
- **Versioned image tags with `imagePullPolicy: IfNotPresent`.** Locally built images are loaded into Minikube rather than pulled from a registry; avoiding `:latest` prevents Kubernetes from trying to pull and failing with `ImagePullBackOff`.
- **Resource requests and limits.** Small requests let the scheduler place Pods predictably on a 3 GB Minikube node, and memory limits stop one Pod from starving the others.
- **SSH tunnel instead of a public port.** Browsers only allow the microphone on `https://` or `localhost`. Tunnelling to `localhost:8080` satisfies that without TLS certificates, and keeps every application port closed in the security group.
- **Minikube with the Docker driver on EC2.** Standard EC2 instances don't support nested virtualization, so VM-based drivers aren't an option; the Docker driver runs the node as a container instead.

---

## Possible Extensions

- HTTPS with the Minikube **ingress** addon and a TLS certificate, so the app works without an SSH tunnel
- Push the image to **Docker Hub** or **Amazon ECR** instead of `minikube image load`
- A **HorizontalPodAutoscaler** driven by metrics-server
- **Helm** chart or **Kustomize** overlays for dev/prod configurations
- A **GitHub Actions** pipeline that builds the image and validates `deploy.yaml` on every push
- Move from Minikube to a multi-node cluster (**kind**, **kubeadm** or **Amazon EKS**)

---

## Tech Stack

`HTML` · `Tailwind CSS` · `JavaScript (Web Speech API)` · `nginx` · `Docker` · `Kubernetes` · `Minikube` · `kubectl` · `AWS EC2` · `Ubuntu 24.04`

## License

MIT
