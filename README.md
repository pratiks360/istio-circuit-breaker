<p align="center">
  <h1 align="center">⚡ istio-circuit-breaker</h1>
  <p align="center">
    <strong>A Flask-based test service for demonstrating Istio circuit breaker patterns on Kubernetes.</strong>
  </p>
  <p align="center">
    <img src="https://img.shields.io/badge/Built_With-Python-3776AB?style=for-the-badge&logo=python&logoColor=white" alt="Python">
    <img src="https://img.shields.io/badge/Framework-Flask-000000?style=for-the-badge&logo=flask&logoColor=white" alt="Flask">
    <img src="https://img.shields.io/badge/Service_Mesh-Istio-466BB0?style=for-the-badge&logo=istio&logoColor=white" alt="Istio">
    <img src="https://img.shields.io/badge/Orchestration-Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white" alt="Kubernetes">
    <img src="https://img.shields.io/badge/Container-Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker">
    <img src="https://img.shields.io/github/last-commit/pratiks360/istio-circuit-breaker?style=for-the-badge&label=Last+Commit" alt="Last Commit">
  </p>
</p>

---

## 📋 Table of Contents

- [What Is This?](#-what-is-this)
- [Features](#-features)
- [How It Works](#-how-it-works)
- [API Endpoints](#-api-endpoints)
- [Quick Start](#-quick-start)
- [Kubernetes & Istio Setup](#-kubernetes--istio-setup)
- [Tech Stack](#️-tech-stack)
- [Troubleshooting](#-troubleshooting)
- [Contributing](#-contributing)
- [License](#-license)

---

## ✨ What Is This?

**istio-circuit-breaker** is a deliberately simple Python/Flask microservice designed to be the *target* of Istio circuit breaker experiments on Kubernetes. It exposes multiple endpoints that simulate different HTTP error conditions and configurable response delays — letting you test how Istio's `DestinationRule` outlier detection and circuit breaking kicks in under real failure scenarios.

> 🔧 **Use case:** Pair this service with an Istio `DestinationRule` and a load-testing tool (e.g. Fortio) to observe how the service mesh protects your cluster from cascading failures.

---

## 🎯 Features

| Feature | Description |
|---|---|
| ✅ **Health Check Endpoint** | Simple liveness probe at `/circuit` to confirm the service is running |
| 💚 **200 OK Simulation** | `/circuit/good` returns a clean HTTP 200 for baseline traffic testing |
| 💥 **500 Internal Error** | `/circuit/internal` triggers server-side error responses for fault injection |
| ⏱️ **Gateway Timeout (504)** | `/circuit/gateway` simulates upstream gateway timeouts |
| 🚫 **503 Service Unavailable** | `/circuit/service` mimics a service being temporarily down |
| 🐢 **Configurable Delay** | `/circuit/delay?time=N` sleeps for N seconds — perfect for timeout & retry testing |

---

## 🧩 How It Works

The app acts as a backend service sitting behind the Istio sidecar proxy. When you apply an Istio `DestinationRule` with outlier detection, Istio monitors this service's responses. Hit the error endpoints enough times and Istio will eject the pod from the load balancing pool — the circuit breaks.

```
┌────────────────┐   HTTP Request   ┌──────────────────┐   Proxied   ┌─────────────────────┐
│                │ ───────────────► │                  │ ──────────► │                     │
│  Load Tester   │                  │  Istio Sidecar   │             │  Flask App (port    │
│  (e.g. Fortio) │                  │  (Envoy Proxy)   │             │  9000) /circuit/*   │
│                │ ◄─────────────── │                  │ ◄────────── │                     │
└────────────────┘   Response or    └──────────────────┘  200/500/   └─────────────────────┘
                     Circuit Open!                        503/504/
                                                          delayed
```

1. **Deploy** — The Flask app runs in a Kubernetes pod with an Istio sidecar injected
2. **Apply `DestinationRule`** — Configure outlier detection thresholds (consecutive errors, interval, ejection %)
3. **Generate Traffic** — Hit `/circuit/internal` or `/circuit/service` repeatedly to trigger ejections
4. **Observe** — Watch Istio eject the instance and return `503 upstream connect error` — the circuit is open!

---

## 🔌 API Endpoints

| Endpoint | Method | Response | Use Case |
|---|---|---|---|
| `/circuit` | GET | `200 App is UP!` | Health / liveness check |
| `/circuit/good` | GET | `200 ok you are getting http 200` | Baseline success traffic |
| `/circuit/internal` | GET | `500 error :: internal server` | Fault injection — server error |
| `/circuit/gateway` | GET | `504 error :: Gateway Timeout` | Gateway timeout simulation |
| `/circuit/service` | GET | `503 error :: service unavailable` | Service outage simulation |
| `/circuit/delay?time=N` | GET | `200` after N seconds | Latency / timeout testing |

---

## 🚀 Quick Start

### Option A · Run with Docker

**Prerequisites:** [Docker](https://docs.docker.com/get-docker/)

```bash
git clone https://github.com/pratiks360/istio-circuit-breaker.git
cd istio-circuit-breaker

# Build the image
docker build -t istio-circuit-breaker .

# Run the container
docker run -p 9000:9000 istio-circuit-breaker
```

Test it:

```bash
curl http://localhost:9000/circuit
# → App is UP!

curl http://localhost:9000/circuit/internal
# → error :: internal server  (HTTP 500)

curl "http://localhost:9000/circuit/delay?time=3"
# → returning response after :: 3 secs  (after 3s delay)
```

### Option B · Run Locally (no Docker)

**Prerequisites:** Python 3.x

```bash
git clone https://github.com/pratiks360/istio-circuit-breaker.git
cd istio-circuit-breaker

pip install -r requirements.txt
python main.py
```

The app will be available at `http://localhost:9000`.

---

## ☸️ Kubernetes & Istio Setup

### 1 · Deploy to Kubernetes

```bash
# Apply the Kubernetes manifests
kubectl apply -f k8s/

# Verify the pod is running with an Istio sidecar (should show 2/2 containers)
kubectl get pods
```

### 2 · Apply the Istio DestinationRule

```bash
kubectl apply -f Istio/
```

The `DestinationRule` configures outlier detection — for example, ejecting a host after a configured number of consecutive 5xx errors within a defined interval.

### 3 · Generate Traffic to Trip the Breaker

Use [Fortio](https://github.com/fortio/fortio) or `curl` in a loop:

```bash
# Hit the error endpoint repeatedly to trigger circuit breaking
for i in $(seq 1 20); do curl http://<SERVICE_IP>/circuit/internal; done
```

Watch Istio's Envoy sidecar stats or Kiali dashboard to see the outlier ejection in action.

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| **Application** | Python 3, Flask, flask-cors |
| **Container** | Docker (python:3 base image) |
| **Orchestration** | Kubernetes |
| **Service Mesh** | Istio (DestinationRule, OutlierDetection) |
| **Port** | `9000` |

---

## 🐛 Troubleshooting

**Q: The pod starts but I don't see an Istio sidecar (`1/1` instead of `2/2`).**  
A: Make sure Istio sidecar injection is enabled for your namespace: `kubectl label namespace default istio-injection=enabled`, then redeploy.

**Q: The `/circuit/delay` endpoint isn't returning a timeout error from Istio.**  
A: Check your `VirtualService` timeout settings. Istio only enforces timeouts if a `VirtualService` with a `timeout` field is applied to the route.

**Q: `docker build` fails with pip errors.**  
A: The base image is `python:3` which pulls the latest Python 3. If you hit dependency conflicts, pin to a specific tag like `python:3.11-slim` in the Dockerfile.

**Q: How do I reset an ejected pod?**  
A: Ejections are temporary and auto-expire based on `baseEjectionTime` in your `DestinationRule`. You can also delete and redeploy the pod to get it back into rotation immediately.

---

## 🤝 Contributing

This project is primarily a learning/demo resource, but PRs and issues are welcome! Feel free to open an issue to suggest new failure-simulation endpoints or improved Istio manifests.

---

## 📄 License

This project is open source under the [MIT License](LICENSE).

---

<p align="center">
  <sub>Built with ❤️ for anyone learning Istio resilience patterns and service mesh observability.</sub>
</p>
