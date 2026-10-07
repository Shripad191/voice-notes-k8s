# Voice Notes on Kubernetes — Minikube on AWS EC2 (Docker + nginx + Kubernetes)

A browser-based voice-to-text notes app, containerized with Docker, served by nginx, and deployed as a replicated workload on a **Kubernetes cluster running in Minikube on a single AWS EC2 instance**.

Open the app, click **Start Dictation**, and your speech is transcribed live into an editable notes area that can be downloaded as a `.txt` file. The app itself is static; the focus of this project is running it the way real services run on Kubernetes: replicated Pods, a stable Service, health checks, rolling updates, and self-healing.

---

## Architecture

```
   Laptop (Chrome)
        │  http://localhost:8080
        │  SSH tunnel  (ssh -L 8080:localhost:8080 ubuntu@<EC2-IP>)
        ▼
 ┌────────────────────── AWS EC2 (Ubuntu 24.04, c7i-flex.large) ──────────────────────┐
 │                                                                                     │
 │   kubectl port-forward svc/voice-notes 8080:80                                      │
 │        │                                                                            │
 │   ┌────▼──────────── Minikube node (Docker container, 192.168.49.2) ────────────┐   │
 │   │                                                                              │   │
 │   │   Service: voice-notes ──selector app=voice-notes──▶ Pod 1: nginx + app      │   │
 │   │   (NodePort 30090 → :80)                         └──▶ Pod 2: nginx + app      │   │
 │   │                                 managed by Deployment: voice-notes (2 replicas) │
 │   └──────────────────────────────────────────────────────────────────────────────┘   │
 └─────────────────────────────────────────────────────────────────────────────────────┘
```

| Component | Role |
|---|---|
| **AWS EC2 (`c7i-flex.large`)** | Ubuntu 24.04 host with 2 vCPU / 4 GiB RAM that runs Docker and Minikube |
| **Minikube (Docker driver)** | Single-node Kubernetes cluster running as a Docker container on the instance |
| **Docker image `voice-notes:v1`** | `nginx:1.27-alpine` serving the app's `index.html` on port 80 |
| **Deployment `voice-notes`** | Keeps 2 replicas running, performs rolling updates, keeps revision history |
| **Service `voice-notes`** | Stable virtual IP and DNS name that load-balances across Ready Pods via the `app=voice-notes` label |
| **Readiness probe** | `GET /` on port 80; a Pod only receives traffic once nginx is serving |
| **SSH tunnel + `kubectl port-forward`** | Delivers the app to the laptop as `http://localhost:8080`, the secure context the browser's microphone API requires |

**Screenshot:** `docs/screenshots/01-ec2-instance.png`
*The EC2 instance in the AWS console: state Running, instance type `c7i-flex.large`, status checks passed.*

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

- An AWS account and an EC2 instance running **Ubuntu 24.04**: this project uses **`c7i-flex.large`** (2 vCPU, 4 GiB RAM) with a 30 GB gp3 root volume. Minikube needs at least 2 vCPUs and 2 GB of free memory, so micro instances are too small
- A security group allowing **SSH (22)** from your IP only
- On the instance: **Docker**, **kubectl** and **Minikube**, run as the `ubuntu` user (the Docker driver refuses to run as root)
- On your laptop: an SSH client, your `.pem` key, and **Chrome or Edge** (the Web Speech API is not available in Firefox)

---

## Set Up the Cluster

```bash
minikube start --driver=docker --cpus=2 --memory=3000mb
minikube addons enable metrics-server
```

**Screenshot:** `docs/screenshots/02-minikube-start.png`
*Minikube starting with the Docker driver and configuring the cluster.*

```bash
minikube status
kubectl get nodes -o wide
```

**Screenshot:** `docs/screenshots/03-minikube-status.png`
*Host, kubelet and API server Running; the single `minikube` node is `Ready`.*

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
kubectl get deploy,pods,svc -o wide
```

This builds the nginx image, loads it into the Minikube node, and creates a Deployment with two replicas plus a NodePort Service in front of them.

**Screenshot:** `docs/screenshots/04-k8s-resources.png`
*The Deployment at 2/2, two Running Pods, and the `voice-notes` NodePort Service.*

---

## Access and Test

**Check the Service is wired to the Pods (on the EC2 instance):**
```bash
kubectl get endpointslices -l kubernetes.io/service-name=voice-notes
curl -s $(minikube service voice-notes --url) | grep "<title>"
```
The EndpointSlice lists two Pod IPs, and `curl` returns `<title>Voice Notes</title>`.

**Open the app from your laptop.** On the laptop:
```bash
ssh -i <your-key>.pem -L 8080:localhost:8080 ubuntu@<EC2-PUBLIC-IP>
```
Inside that SSH session:
```bash
kubectl port-forward svc/voice-notes 8080:80
```
Then browse to `http://localhost:8080`, click **Start Dictation**, allow microphone access, and speak.

**Screenshot:** `docs/screenshots/05-app-dictation.png`
*The app at `localhost:8080` with status "Listening..." and live transcribed text in the notes area.*

---

## Self-Healing

Delete one Pod and the Deployment's ReplicaSet immediately creates a replacement to restore the desired count of 2:

```bash
kubectl get pods -o wide
kubectl delete pod <one-pod-name>
kubectl get pods -o wide
kubectl get events --sort-by=.metadata.creationTimestamp | grep -E "Killing|SuccessfulCreate|Scheduled|Started" | tail -6
```

**Screenshot:** `docs/screenshots/06-self-healing.png`
*Still 2 Pods after the deletion: one with a new name and an age of a few seconds, plus the events showing the old Pod killed and the new one created.*

Restart the `port-forward` afterwards, since it was attached to the original Pod.

---

## Update to a New Version

```bash
docker build -t voice-notes:v2 .
minikube image load voice-notes:v2
kubectl set image deploy/voice-notes web=voice-notes:v2
kubectl rollout status deploy/voice-notes

# Roll back if needed
kubectl rollout undo deploy/voice-notes
```

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
| **EC2 `c7i-flex.large`** | Free Tier eligible on newer AWS accounts; otherwise billed per hour while **running**. Check the Free Tier page in the Billing console for your account |
| **EBS gp3 (30 GB)** | Billed (or counted against Free Tier storage) while the volume exists, including when the instance is stopped |
| **Data transfer** | Negligible; traffic goes through the SSH session |

Stopping the instance between sessions is the biggest saving. Minikube and all Kubernetes objects survive a stop/start; just run `minikube start` again.

---

## Design Decisions

- **nginx on Alpine to serve a static app.** The app is a single HTML file, so a small, well-known web server image keeps the container lightweight and fast to roll out.
- **Deployment with 2 replicas, not a bare Pod.** Gives self-healing, rolling updates and rollback history, and demonstrates load-balancing across Pods via the Service.
- **Labels and selectors as the only link between objects.** The Deployment's `matchLabels`, the Pod template labels and the Service selector are kept identical (`app=voice-notes`).
- **Readiness probe on `/`.** New or restarted Pods only receive traffic once nginx is serving, which keeps rolling updates zero-downtime.
- **Versioned image tags with `imagePullPolicy: IfNotPresent`.** Locally built images are loaded into Minikube rather than pulled from a registry; avoiding `:latest` prevents `ImagePullBackOff`.
- **Resource requests and limits.** Small requests let the scheduler place Pods predictably, and memory limits stop one Pod from starving the others.
- **SSH tunnel instead of a public port.** Browsers only allow the microphone on `https://` or `localhost`. Tunnelling to `localhost:8080` satisfies that without TLS certificates and keeps every application port closed in the security group.
- **Minikube with the Docker driver on EC2.** Standard EC2 instances don't support nested virtualization, so the Docker driver runs the Kubernetes node as a container instead of a VM.

---

## Possible Extensions

- HTTPS with the Minikube **ingress** addon and a TLS certificate, so the app works without an SSH tunnel
- Push the image to **Docker Hub** or **Amazon ECR** instead of `minikube image load`
- A **HorizontalPodAutoscaler** driven by metrics-server
- A **GitHub Actions** pipeline that builds the image and validates `deploy.yaml` on every push
- Move to a multi-node cluster (**kind**, **kubeadm** or **Amazon EKS**)

---

## Tech Stack

`HTML` · `Tailwind CSS` · `JavaScript (Web Speech API)` · `nginx` · `Docker` · `Kubernetes` · `Minikube` · `kubectl` · `AWS EC2` · `Ubuntu 24.04`

## License

MIT
