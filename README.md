# 🎙️ Voice Notes on Kubernetes (Minikube on AWS EC2)

A browser-based **voice-to-text notes app** containerized with **Docker**, served by **nginx**, and deployed on a **Kubernetes** cluster running in **Minikube** on an **AWS EC2** instance.

Speak into your microphone and watch your words appear in real time. Download your notes as a `.txt` file with one click.

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![Minikube](https://img.shields.io/badge/Minikube-326CE5?logo=kubernetes&logoColor=white)
![AWS EC2](https://img.shields.io/badge/AWS%20EC2-FF9900?logo=amazonec2&logoColor=white)
![nginx](https://img.shields.io/badge/nginx-009639?logo=nginx&logoColor=white)

---

## 📌 Table of Contents

- [Features](#-features)
- [Tech Stack](#-tech-stack)
- [Architecture](#-architecture)
- [Project Structure](#-project-structure)
- [Prerequisites](#-prerequisites)
- [Deployment Steps](#-deployment-steps)
- [Accessing the App](#-accessing-the-app)
- [Updating and Rolling Back](#-updating-and-rolling-back)
- [Screenshots](#-screenshots)
- [Troubleshooting](#-troubleshooting)
- [Cleanup](#-cleanup)
- [What I Learned](#-what-i-learned)

---

## ✨ Features

- 🎤 **Live dictation** using the browser's Web Speech API (continuous + interim results)
- 📝 Editable notes area. Dictation appends to existing text
- 💾 **Download** notes as `voice_notes.txt`
- 🧹 **Clear** notes with one click
- ☸️ Runs as a **Kubernetes Deployment with 2 replicas** behind a **Service**
- ❤️ **Readiness probe** so traffic only goes to healthy Pods
- 🔄 **Rolling updates and rollbacks** with zero downtime
- 📦 Lightweight image based on `nginx:1.27-alpine`

---

## 🛠 Tech Stack

| Layer | Technology |
| --- | --- |
| Frontend | HTML, Tailwind CSS (CDN), JavaScript, Web Speech API |
| Web server | nginx 1.27 (Alpine) |
| Containerization | Docker |
| Orchestration | Kubernetes (Minikube, Docker driver) |
| Cloud | AWS EC2 (Ubuntu 24.04, t3.medium) |
| CLI tools | kubectl, minikube, docker |

---

## 🏗 Architecture

```
 Your laptop (Chrome)
        │  http://localhost:8080   (SSH tunnel: -L 8080:localhost:8080)
        ▼
 ┌──────────────────────── AWS EC2 instance (Ubuntu) ────────────────────────┐
 │                                                                            │
 │   kubectl port-forward svc/voice-notes 8080:80                             │
 │        │                                                                   │
 │   ┌────▼──────────── Minikube (Docker container, 192.168.49.2) ─────────┐  │
 │   │                                                                      │  │
 │   │   Service: voice-notes (NodePort 30090, port 80)                     │  │
 │   │        │  selector: app=voice-notes                                  │  │
 │   │        ├──────────────► Pod 1: nginx + index.html (:80)              │  │
 │   │        └──────────────► Pod 2: nginx + index.html (:80)              │  │
 │   │                         (managed by Deployment: voice-notes, 2 replicas) │
 │   └──────────────────────────────────────────────────────────────────────┘  │
 └────────────────────────────────────────────────────────────────────────────┘
```

> **Why an SSH tunnel?** Browsers only allow microphone access on secure origins (`https://` or `localhost`). Tunnelling makes the app reachable at `http://localhost:8080`, so dictation works without setting up HTTPS.

---

## 📁 Project Structure

```
voice-notes-k8s/
├── index.html        # The Voice Notes web app
├── Dockerfile        # nginx image that serves index.html
├── deploy.yaml       # Kubernetes Deployment + Service
├── screenshots/      # Proof-of-concept screenshots used in this README
├── .gitignore
└── README.md
```

---

## ✅ Prerequisites

- AWS EC2 instance: **Ubuntu 24.04**, at least **t3.medium** (2 vCPU, 4 GB RAM), **30 GB** disk
- Security group: **SSH (22)** allowed from your IP
- Installed on the instance: **Docker**, **kubectl**, **minikube**
- A Chromium-based browser (Chrome / Edge) on your laptop for the Speech API

---

## 🚀 Deployment Steps

### 1. Start Minikube

```bash
minikube start --driver=docker --cpus=2 --memory=3000mb
minikube addons enable metrics-server
kubectl get nodes
```

### 2. Clone this repository

```bash
git clone https://github.com/<your-username>/voice-notes-k8s.git
cd voice-notes-k8s
```

### 3. Build the image and load it into Minikube

```bash
docker build -t voice-notes:v1 .
minikube image load voice-notes:v1
minikube image ls | grep voice-notes
```

> Minikube has its own image store, so locally built images must be loaded with `minikube image load`. The manifest uses `imagePullPolicy: IfNotPresent` so Kubernetes uses the loaded image instead of pulling from Docker Hub.

### 4. Deploy to Kubernetes

```bash
kubectl apply -f deploy.yaml --dry-run=server   # validate first
kubectl apply -f deploy.yaml
kubectl get pods -w
kubectl get deploy,svc
```

### 5. Verify from inside the EC2 instance

```bash
kubectl get endpointslices -l kubernetes.io/service-name=voice-notes
curl -s $(minikube service voice-notes --url) | grep "<title>"
# <title>Voice Notes</title>
```

---

## 🌐 Accessing the App

**On your laptop**, open an SSH session with a tunnel:

```bash
ssh -i <your-key>.pem -L 8080:localhost:8080 ubuntu@<EC2-PUBLIC-IP>
```

**Inside that SSH session**, forward the Service:

```bash
kubectl port-forward svc/voice-notes 8080:80
```

**On your laptop**, open 👉 **http://localhost:8080** in Chrome, click **Start Dictation**, and allow microphone access.

---

## 🔄 Updating and Rolling Back

```bash
# Edit index.html, then build and load a new version
docker build -t voice-notes:v2 .
minikube image load voice-notes:v2

# Rolling update
kubectl set image deploy/voice-notes web=voice-notes:v2
kubectl rollout status deploy/voice-notes

# History and rollback
kubectl rollout history deploy/voice-notes
kubectl rollout undo deploy/voice-notes
```

Restart the `port-forward` after a rollout, because it stays attached to the old Pods.

Other useful operations:

```bash
kubectl scale deploy/voice-notes --replicas=4     # scale out
kubectl delete pod -l app=voice-notes             # self-healing: Pods are recreated
kubectl logs -l app=voice-notes --prefix          # nginx access logs
kubectl top pods                                  # resource usage
```

---

## 📸 Screenshots

| # | What it shows | Screenshot |
| --- | --- | --- |
| 1 | EC2 instance running (type, state) | ![EC2](screenshots/01-ec2-instance.png) |
| 2 | Minikube running and node Ready | ![Minikube status](screenshots/02-minikube-status.png) |
| 3 | Docker image built and loaded into Minikube | ![Image](screenshots/03-image-build-load.png) |
| 4 | Deployment, Pods and Service running | ![Kubernetes resources](screenshots/04-k8s-resources.png) |
| 5 | Service endpoints and curl test | ![Endpoints](screenshots/05-endpoints-curl.png) |
| 6 | SSH tunnel + port-forward active | ![Port forward](screenshots/06-port-forward.png) |
| 7 | App open in the browser | ![App UI](screenshots/07-app-ui.png) |
| 8 | Live dictation with transcribed text | ![Dictation](screenshots/08-dictation-working.png) |
| 9 | Downloaded `voice_notes.txt` | ![Download](screenshots/09-download.png) |
| 10 | Self-healing / rolling update | ![Self healing](screenshots/10-self-healing.png) |

---

## 🧯 Troubleshooting

| Problem | Cause | Fix |
| --- | --- | --- |
| `unknown field "spec.repllicas"` | Typo in YAML field names | Fix spelling (`replicas`, `matchLabels`, `httpGet`); validate with `--dry-run=server` |
| `SVC_UNREACHABLE: no running pod for service` | Service selector doesn't match Pod labels | Make `selector`, `matchLabels` and template `labels` identical |
| `ImagePullBackOff` | Image not loaded into Minikube | `minikube image load voice-notes:v1` |
| `Error: not-allowed` in the app | Page not on a secure origin / mic blocked | Use `http://localhost:8080` via SSH tunnel; allow mic in Chrome |
| "Speech Recognition API is not supported" | Firefox / Safari | Use Chrome or Edge |
| `bind: address already in use` | Another port-forward holds 8080 | `pkill -f port-forward` |

---

## 🧹 Cleanup

```bash
kubectl delete -f deploy.yaml
minikube image rm voice-notes:v1
minikube stop        # or: minikube delete
```

Stop (or terminate) the EC2 instance when you're done to avoid charges.

---

## 📚 What I Learned

- Containerizing a static web app with nginx
- Writing Kubernetes **Deployment** and **Service** manifests from scratch
- How **labels and selectors** connect Services to Pods (and how a mismatch breaks routing)
- Loading local images into Minikube and using `imagePullPolicy`
- Health checks with **readiness probes**
- **Rolling updates, rollbacks, scaling and self-healing**
- Why browser APIs like the microphone need a **secure context**, and using SSH tunnels to provide one
- Running and managing Minikube on a cloud VM (EC2)

---

## 👤 Author

**<Your Name>**
- GitHub: [@<your-username>](https://github.com/<your-username>)
- LinkedIn: [<your-linkedin>](https://www.linkedin.com/in/<your-linkedin>)

⭐ If you found this project helpful, give it a star!
