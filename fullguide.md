# ChaosController × Sock Shop — Second Target App Demo Guide

> **Prerequisites:** You have already completed the [COMPLETE_JENKINS_DEMO_GUIDE.md](./COMPLETE_JENKINS_DEMO_GUIDE.md). Jenkins, ChaosController image, and all plugins are already installed and working.
> **Goal:** Add a second, heavier target application (Sock Shop — 13 microservices with authentication) to demonstrate framework portability.

---

## 1. About the Sock Shop

### Weaveworks Sock Shop — Cloud-Native Microservices Demo

| Detail | Value |
|--------|-------|
| **Repository** | <https://github.com/microservices-demo/microservices-demo> |
| **License** | Apache-2.0 |
| **Why this app?** | 13 microservices, user authentication, shopping cart, orders, payments, multiple databases (MySQL, MongoDB), message queue (RabbitMQ), polyglot (Java, Go, Node.js) |

**Architecture (13 services):**

```
                    ┌──────────────┐
                    │  edge-router │  (Nginx — port 8083)
                    │   (Nginx)    │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  front-end   │  (Node.js — Web UI)
                    └──┬───┬───┬──┘
                       │   │   │
        ┌──────────────┤   │   ├──────────────┐
        │              │   │   │              │
  ┌─────▼─────┐ ┌─────▼───▼─┐ ┌─────▼─────┐ ┌──▼──────┐
  │ catalogue │ │   carts   │ │  orders   │ │  user   │
  │   (Go)    │ │  (Java)   │ │  (Java)   │ │  (Go)   │
  └─────┬─────┘ └─────┬─────┘ └──┬──┬──┬──┘ └────┬────┘
        │             │          │  │  │          │
  ┌─────▼─────┐ ┌─────▼─────┐   │  │  │    ┌────▼────┐
  │catalogue- │ │  carts-db │   │  │  │    │ user-db │
  │    db     │ │ (MongoDB) │   │  │  │    │(MongoDB)│
  │  (MySQL)  │ └───────────┘   │  │  │    └─────────┘
  └───────────┘                 │  │  │
                          ┌─────▼┐ │ ┌▼────────┐
                          │ship- │ │ │ payment │
                          │ping  │ │ │  (Go)   │
                          │(Java)│ │ └─────────┘
                          └──┬───┘ │
                             │   ┌─▼──────────┐
                        ┌────▼─┐ │queue-master│
                        │Rabbit│ │   (Java)   │
                        │  MQ  │ └────────────┘
                        └──────┘
```

**Key features for your presentation:**
- **User Authentication** — register/login with the `user` service
- **Shopping Cart** — add socks, manage cart via `carts` service
- **Order Processing** — place orders via `orders` → `shipping` → `payment` chain
- **Product Catalogue** — browse socks via `catalogue` + MySQL database
- **Message Queue** — RabbitMQ for async shipping notifications

---

## 2. Port Allocation (No Conflicts)

| Service | Port | Notes |
|---------|------|-------|
| **Jenkins** | `8080` | Already running |
| **ChaosController** | `5050` | Started by pipeline |
| **Sock Shop (Web UI)** | `8083` | Edge-router mapped here |
| **Voting App (Vote)** | `8082` | From previous demo (if still running) |
| **Voting App (Results)** | `8081` | From previous demo (if still running) |

> **Important:** Stop the voting app first if it's running: `cd ~/voting-app-chaos-demo && docker-compose down`

---

## 3. Fork, Clone & Prepare the Sock Shop

### Step 1: Fork on GitHub

1. Open <https://github.com/microservices-demo/microservices-demo> in your browser
2. Click **Fork** (top-right) → **Create fork**
3. You now have `https://github.com/YOUR_USERNAME/microservices-demo`

### Step 2: Clone YOUR Fork

```bash
cd ~
git clone https://github.com/YOUR_USERNAME/microservices-demo.git sockshop-chaos-demo
cd sockshop-chaos-demo
```

### Step 3: Create Wrapper Dockerfiles for Chaos-Testable Services

The Sock Shop uses pre-built Docker images. We need to create thin wrapper Dockerfiles that install `iproute2` (required for network fault injection like latency and packet loss).

> **Note:** The Java-based services (carts, orders, shipping) and Node.js front-end use very old Alpine Linux (v3.4) base images where `apk` is used. `stress-ng` (CPU/memory stress) is not available in Alpine 3.4, so only network faults are supported for these services. Go-based services (catalogue, payment, user) are built FROM scratch and cannot have tools installed — they support **container crash** tests only.

Create the directory structure:

```bash
mkdir -p chaos-dockerfiles/front-end
mkdir -p chaos-dockerfiles/carts
mkdir -p chaos-dockerfiles/orders
mkdir -p chaos-dockerfiles/shipping
```

**Front-end wrapper (Node.js/Alpine):**

```bash
cat > chaos-dockerfiles/front-end/Dockerfile << 'EOF'
FROM weaveworksdemos/front-end:0.3.12
USER root
RUN apk add --no-cache iproute2 || true
USER node
EOF
```

**Carts wrapper (Java/Alpine):**

```bash
cat > chaos-dockerfiles/carts/Dockerfile << 'EOF'
FROM weaveworksdemos/carts:0.4.8
USER root
RUN apk add --no-cache iproute2 || true
EOF
```

**Orders wrapper (Java/Alpine):**

```bash
cat > chaos-dockerfiles/orders/Dockerfile << 'EOF'
FROM weaveworksdemos/orders:0.4.7
USER root
RUN apk add --no-cache iproute2 || true
EOF
```

**Shipping wrapper (Java/Alpine):**

```bash
cat > chaos-dockerfiles/shipping/Dockerfile << 'EOF'
FROM weaveworksdemos/shipping:0.4.8
USER root
RUN apk add --no-cache iproute2 || true
EOF
```

### Step 4: Create the Modified docker-compose.yml

Replace whatever exists in `deploy/docker-compose/docker-compose.yml` (or create fresh at the repo root):

```bash
cat > docker-compose.yml << 'COMPOSE_EOF'
services:
  front-end:
    build: ./chaos-dockerfiles/front-end
    hostname: front-end
    restart: always
    cap_add:
      - NET_ADMIN
    privileged: true
    environment:
      - reschedule=on-node-failure

  edge-router:
    image: weaveworksdemos/edge-router:0.1.1
    ports:
      - '8083:80'
    cap_add:
      - NET_BIND_SERVICE
      - CHOWN
      - SETGID
      - SETUID
      - DAC_OVERRIDE
    tmpfs:
      - /var/run:rw,noexec,nosuid
    hostname: edge-router
    restart: always

  catalogue:
    image: weaveworksdemos/catalogue:0.3.5
    hostname: catalogue
    restart: always
    cap_add:
      - NET_BIND_SERVICE

  catalogue-db:
    image: weaveworksdemos/catalogue-db:0.3.0
    hostname: catalogue-db
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=fake_password
      - MYSQL_ALLOW_EMPTY_PASSWORD=true
      - MYSQL_DATABASE=socksdb

  carts:
    build: ./chaos-dockerfiles/carts
    hostname: carts
    restart: always
    cap_add:
      - NET_ADMIN
      - NET_BIND_SERVICE
    privileged: true
    tmpfs:
      - /tmp:rw,noexec,nosuid
    environment:
      - JAVA_OPTS=-Xms64m -Xmx128m -XX:+UseG1GC -Djava.security.egd=file:/dev/urandom -Dspring.zipkin.enabled=false

  carts-db:
    image: mongo:3.4
    hostname: carts-db
    restart: always
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
    tmpfs:
      - /tmp:rw,noexec,nosuid

  orders:
    build: ./chaos-dockerfiles/orders
    hostname: orders
    restart: always
    cap_add:
      - NET_ADMIN
      - NET_BIND_SERVICE
    privileged: true
    tmpfs:
      - /tmp:rw,noexec,nosuid
    environment:
      - JAVA_OPTS=-Xms64m -Xmx128m -XX:+UseG1GC -Djava.security.egd=file:/dev/urandom -Dspring.zipkin.enabled=false

  orders-db:
    image: mongo:3.4
    hostname: orders-db
    restart: always
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
    tmpfs:
      - /tmp:rw,noexec,nosuid

  shipping:
    build: ./chaos-dockerfiles/shipping
    hostname: shipping
    restart: always
    cap_add:
      - NET_ADMIN
      - NET_BIND_SERVICE
    privileged: true
    tmpfs:
      - /tmp:rw,noexec,nosuid
    environment:
      - JAVA_OPTS=-Xms64m -Xmx128m -XX:+UseG1GC -Djava.security.egd=file:/dev/urandom -Dspring.zipkin.enabled=false

  queue-master:
    image: weaveworksdemos/queue-master:0.3.1
    hostname: queue-master
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: always
    cap_add:
      - NET_BIND_SERVICE
    tmpfs:
      - /tmp:rw,noexec,nosuid

  rabbitmq:
    image: rabbitmq:3.6.8
    hostname: rabbitmq
    restart: always
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
      - DAC_OVERRIDE

  payment:
    image: weaveworksdemos/payment:0.4.3
    hostname: payment
    restart: always
    cap_add:
      - NET_BIND_SERVICE

  user:
    image: weaveworksdemos/user:0.4.4
    hostname: user
    restart: always
    cap_add:
      - NET_BIND_SERVICE
    environment:
      - MONGO_HOST=user-db:27017

  user-db:
    image: weaveworksdemos/user-db:0.4.0
    hostname: user-db
    restart: always
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
    tmpfs:
      - /tmp:rw,noexec,nosuid

networks:
  default:
    name: sockshop-chaos-demo_default
COMPOSE_EOF
```

> **What changed vs original Sock Shop:**
> | Change | Why |
> |--------|-----|
> | `build:` for front-end, carts, orders, shipping | Wrapper Dockerfiles that add `iproute2` |
> | `cap_add: NET_ADMIN` + `privileged: true` on testable services | Required for `tc` and `iptables` fault injection |
> | Removed `read_only: true` from built services | Allows ChaosController to install/run tools |
> | Removed `cap_drop: all` from built services | Was blocking network admin capabilities |
> | Port `8083:80` for edge-router | Avoids conflict with Jenkins on 8080 |
> | Explicit network name | Ensures consistent naming for Jenkins pipeline |

### Step 5: Build and Test Locally

```bash
cd ~/sockshop-chaos-demo
docker compose up -d --build
```

> **First build will take 2-3 minutes** (downloading base images + installing tools).

Wait ~60 seconds for all services to start, then verify:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

You should see 14 containers running.

**Check the web UI:**

| URL | What You See |
|-----|-------------|
| <http://localhost:8083> | Sock Shop storefront — browse socks, register, login, add to cart |

**Test user authentication:**
1. Click **Login** (top right) — use default credentials: `user` / `password`
2. Or click **Register** to create a new account
3. Browse the catalogue, add socks to your cart, place an order

**Verify chaos tools were installed:**

```bash
docker exec sockshop-chaos-demo-front-end-1 which tc
# Should print: /sbin/tc or /usr/sbin/tc

docker exec sockshop-chaos-demo-carts-1 which tc
# Should print: /sbin/tc
```

> ✅ If the storefront loads AND `tc` commands return paths, the app is ready.

### Step 6: Quick Manual Test with ChaosController

```bash
docker rm -f chaos-test 2>/dev/null || true
docker run -d \
  --name chaos-test \
  --network sockshop-chaos-demo_default \
  -p 5050:5050 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e GEMINI_API_KEY=YOUR_KEY_HERE \
  chaos-controller:latest
```

1. Open <http://localhost:5050> — ChaosController dashboard
2. You should see all 14 Sock Shop containers discovered
3. Inject a **latency fault** on the `front-end` service (e.g., 2000ms)
4. Open <http://localhost:8083> — the page should be noticeably slower
5. **Recover** in the dashboard
6. **Run an experiment** — Probe URL: `http://front-end:8079/` (internal port)

> **Probe URLs for Sock Shop services:**
> | Service | Probe URL |
> |---------|-----------|
> | front-end | `http://front-end:8079/` |
> | carts | `http://carts/health` |
> | orders | `http://orders/health` |
> | shipping | `http://shipping/health` |
> | catalogue | `http://catalogue/health` |
> | payment | `http://payment/health` |
> | user | `http://user/health` |

**Cleanup after testing:**

```bash
docker stop chaos-test && docker rm chaos-test
cd ~/sockshop-chaos-demo && docker compose down
```

---

## 4. Create the Jenkins Pipeline Files

### Step 1: Create chaos-config.yml

```bash
cd ~/sockshop-chaos-demo

cat > chaos-config.yml << 'CONFIG_EOF'
version: "1.0"
project: "sockshop-chaos-demo"

controller:
  url: "http://localhost:5050"
  timeout_seconds: 300

thresholds:
  latency_regression_percent: 50
  max_5xx_rate_percent: 10
  max_payload_crash_rate_percent: 0

services:
  - name: front-end
    probe_url: "http://front-end:8079/"

  - name: carts
    probe_url: "http://carts/health"

  - name: orders
    probe_url: "http://orders/health"

tests:
  - id: "latency-frontend"
    type: latency
    target: front-end
    params:
      delay_ms: 2000
      num_requests: 5
    on_fail: block

  - id: "latency-orders"
    type: latency
    target: orders
    params:
      delay_ms: 3000
      num_requests: 5
    on_fail: warn

report:
  output_path: "chaos-report.html"
  json_path: "chaos-report.json"
  ai_summary: true
  fail_on: "block"
CONFIG_EOF
```

### Step 2: Create the Jenkinsfile

```bash
cat > Jenkinsfile << 'JENKINS_EOF'
pipeline {
    agent any

    environment {
        GEMINI_API_KEY       = credentials('gemini-api-key')
        COMPOSE_PROJECT_NAME = 'sockshop-chaos-demo'
        COMPOSE_FILE         = 'docker-compose.yml'
        APP_NETWORK          = 'sockshop-chaos-demo_default'
        CHAOS_IMAGE          = 'chaos-controller:latest'
        CHAOS_CONTAINER      = 'chaos-controller-sockshop'
    }

    stages {

        stage('Build & Start Sock Shop') {
            steps {
                echo '🔨 Building and starting the Sock Shop stack (13 microservices)...'
                sh "docker-compose -f ${COMPOSE_FILE} up --build -d"
                echo '⏳ Waiting 60s for all services to initialize...'
                sh 'sleep 60'
                echo '✅ Sock Shop stack is up.'
            }
        }

        stage('Start ChaosController') {
            steps {
                echo '🔥 Starting ChaosController...'
                sh "docker rm -f ${CHAOS_CONTAINER} 2>/dev/null || true"
                sh """
                    docker run -d \
                        --name ${CHAOS_CONTAINER} \
                        --network ${APP_NETWORK} \
                        -p 5050:5050 \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -e GEMINI_API_KEY=${GEMINI_API_KEY} \
                        ${CHAOS_IMAGE}
                """
                sh "docker network connect ${APP_NETWORK} jenkins || true"
                sh '''
                    for i in $(seq 1 30); do
                        curl -sf http://chaos-controller-sockshop:5050/status > /dev/null && echo "ChaosController ready!" && exit 0
                        echo "Waiting for ChaosController... ($i/30)"
                        sleep 5
                    done
                    echo "ChaosController failed to start!" && exit 1
                '''
                echo '✅ ChaosController is ready.'
            }
        }

        stage('Application Health Check') {
            steps {
                echo '🏥 Verifying Sock Shop services are healthy...'
                sh '''
                    for i in $(seq 1 20); do
                        curl -sf http://edge-router:80/ > /dev/null && echo "Sock Shop healthy!" && exit 0
                        echo "Waiting for Sock Shop... ($i/20)"
                        sleep 5
                    done
                    echo "Sock Shop failed to become healthy!" && exit 1
                '''
                echo '✅ Sock Shop is healthy and ready for chaos testing.'
            }
        }

        stage('Resilience Testing Gate') {
            steps {
                script {
                    def agentHost = sh(script: "hostname -I | awk '{print \$1}'", returnStdout: true).trim()

                    echo """
╔══════════════════════════════════════════════════════════════╗
║           🔥  RESILIENCE TESTING GATE  🔥                    ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║   Dashboard: http://${agentHost}:5050                        ║
║   Sock Shop: http://${agentHost}:8083                        ║
║                                                              ║
║   13 microservices have been auto-discovered.                ║
║   Open the dashboard to:                                     ║
║     • Inject faults (latency, crash, CPU, packet loss...)   ║
║     • Run chaos experiments on front-end, carts, orders     ║
║     • Get AI-powered remediation reports (Gemini)            ║
║     • Test user authentication under chaos conditions       ║
║                                                              ║
║   When satisfied, come back here and click APPROVE.          ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
                    """

                    input(
                        id: 'ResilienceApproval',
                        message: 'Resilience Testing Complete?',
                        ok: 'Approve — System is resilient, continue to deploy'
                    )

                    echo '✅ Resilience gate APPROVED.'
                }
            }
        }

        stage('Deploy to Production') {
            steps {
                echo '🚀 Resilience gate passed — deploying to production...'
                echo '(In a real scenario, this would run: kubectl apply, helm upgrade, etc.)'
                echo '✅ Deployed successfully!'
            }
        }
    }

    post {
        always {
            echo '🧹 Cleaning up...'
            sh "docker stop ${CHAOS_CONTAINER} 2>/dev/null || true"
            sh "docker rm ${CHAOS_CONTAINER} 2>/dev/null || true"
            sh "docker-compose -f ${COMPOSE_FILE} down 2>/dev/null || true"
        }
        failure {
            echo '❌ Pipeline failed or was rejected.'
        }
        success {
            echo '✅ Pipeline succeeded!'
        }
    }
}
JENKINS_EOF
```

### Step 3: Commit & Push

```bash
cd ~/sockshop-chaos-demo
git add .
git commit -m "Add ChaosController pipeline: Jenkinsfile, chaos-config, wrapper Dockerfiles, modified compose"
git push origin master
```

> **Note:** The Sock Shop repo uses `master` branch (not `main`). Adjust if your fork uses a different default branch.

---

## 5. Add the Pipeline to Jenkins

Since Jenkins is already running from the voting app setup, you just need to add a **second pipeline job**.

### Step 1: Create a New Pipeline Job

1. Open Jenkins at <http://localhost:8080>
2. Click **New Item** (left sidebar)
3. Name: `sockshop-resilience-gate`
4. Select **Pipeline** → **OK**
5. Scroll down to **Pipeline** section:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `https://github.com/YOUR_USERNAME/microservices-demo.git` (your fork!)
   - Branch: `*/master`
6. Click **Save**

> **That's it.** No need to reinstall plugins, credentials, or Docker Compose — they're all still configured from the voting app setup.

### Step 2: Run the Pipeline

1. Click **Build Now**
2. Watch the **Stage View**:
   - ✅ Build & Start Sock Shop (~2-3 min first time for image downloads)
   - ✅ Start ChaosController
   - ✅ Application Health Check
   - ⏸️ **Resilience Testing Gate** (pipeline PAUSES here!)

### Step 3: Do the Chaos Testing

1. Open <http://localhost:5050> — ChaosController dashboard
2. You will see all 14 Sock Shop containers
3. **Suggested demo flow:**
   - Open <http://localhost:8083> — show the Sock Shop working
   - Login with `user` / `password`
   - Add some socks to cart
   - **Inject latency** on the `carts` service (3000ms)
   - Try adding to cart again — observe the delay
   - **Recover** the carts service
   - **Inject latency** on `front-end` — entire site becomes slow
   - **Run a full experiment** on `orders` service
   - **Generate AI report** — show Gemini analysis
   - **Try the AI Autopilot** — let AI plan and execute tests automatically

4. Go back to Jenkins → **Approve** → pipeline deploys ✅

---

## 6. Which Services to Test (Chaos Testing Guide)

### Full Fault Injection Support (latency, stress, packet loss, etc.)

These services have `iproute2` installed via wrapper Dockerfiles (network faults only):

| Service | What it Does | Interesting to Test Because |
|---------|-------------|---------------------------|
| **front-end** | Web UI (Node.js) | Single point of entry — breaking this breaks everything |
| **carts** | Shopping cart (Java) | Tests if orders can still be placed without cart data |
| **orders** | Order processing (Java) | Complex dependency chain: orders → shipping → payment |
| **shipping** | Shipping (Java) | Tests async flow via RabbitMQ |

### Crash-Only Support (container stop/kill)

These use Go scratch images — no package manager available:

| Service | What it Does |
|---------|-------------|
| **catalogue** | Product listings |
| **payment** | Payment processing |
| **user** | Authentication |

> **Tip for presentation:** Crashing the `user` service and showing that login fails while the catalogue still works is a great demo of graceful degradation (or lack thereof).

### 6.1 Feature Compatibility Checklist (What won't work and why)

Because the Sock Shop uses older Alpine Linux images and scratch-based Go images, some specific features in the **ChaosController Frontend** will not function or will fail during execution:

#### ❌ What WILL NOT Work

1. **Manual Faults: CPU Stress & Memory Stress**
   - **Why:** These require the `stress-ng` package. As explained earlier, `stress-ng` is unavailable in the Alpine 3.4 repositories used by the Java/Node.js services, and cannot be installed on the Go scratch images.
   - **What happens:** If you try to inject these via the dashboard, the controller will return an error or the fault will fail silently inside the container.
2. **AI Autopilot (if it generates a CPU/Memory test)**
   - **Why:** The Autopilot (Gemini) generates test plans dynamically. Since it doesn't know `stress-ng` is missing inside the containers, it might suggest a CPU Stress test. 
   - **What happens:** The plan generation will work perfectly, but the *execution* of that specific step will fail.
3. **Network Faults on Go Services (catalogue, user, payment)**
   - **Why:** Network faults (Latency, Packet Loss, Bandwidth, Network Partition, DNS Failure) require `iproute2` and `iptables`. The Go services are built FROM scratch (they contain literally nothing but the Go binary).
   - **What happens:** Injecting latency on the `user` service will throw an error saying `tc` (Traffic Control) is not found.

#### ✅ What WILL Work Perfectly

Everything else works exactly as it did in the voting app:
1. **AI Scenario Generator:** It reads the `docker-compose.yml` and will successfully analyze the 13-service architecture.
2. **AI Predictor (Pre-Flight Check):** Auto-detects all 14 containers (including databases) and accurately maps the dependencies (e.g., edge-router → front-end → carts).
3. **AI Analyst (Post-Experiment Report):** Will successfully analyze any completed network or crash experiment and generate remediation steps.
4. **AI Live Advisor:** Can be triggered during any active fault (e.g., while latency is being injected into the front-end) to assess blast radius.
5. **Cross-Run Trend Comparator:** Works flawlessly by comparing history JSONs.
6. **Manual Faults: Latency & Packet Loss:** Works perfectly on `front-end`, `carts`, `orders`, and `shipping`.
7. **Manual Faults: Container Crash:** Works perfectly on **all** services (including `user` and `catalogue`).

---

## 7. Presentation Talking Points (Sock Shop vs Voting App)

When presenting to your teachers, highlight the **framework portability**:

| Aspect | Voting App | Sock Shop |
|--------|-----------|-----------|
| **Services** | 5 | 13 |
| **Languages** | Python, Node.js, .NET | Java, Go, Node.js |
| **Has Auth?** | ❌ | ✅ (user service + MongoDB) |
| **Databases** | Redis + Postgres | MySQL + MongoDB × 3 + RabbitMQ |
| **Complexity** | Simple queue pipeline | Full e-commerce with cart, orders, payments |
| **Changes needed to our framework** | None | None |
| **Changes needed to target app** | Add iproute2/stress-ng + NET_ADMIN | Same — add iproute2 + NET_ADMIN |

> **Key message:** "We tested our framework on two completely different applications — a simple 5-service voting app and a complex 13-service e-commerce platform with authentication. The framework required ZERO code changes. The only modifications were to the target application's Docker configuration to enable Linux networking tools."

---

## 8. Troubleshooting

| Problem | Solution |
|---------|----------|
| Port 8083 not accessible | Check `docker ps` — edge-router should show `0.0.0.0:8083->80/tcp` |
| `mongo:3.4` image pull fails | The image is old. Try replacing `mongo:3.4` with `mongo:4.4` in docker-compose.yml |
| Carts/Orders show errors | Java services need ~45-60 seconds to fully start. Wait and retry. |
| ChaosController can't find containers | Verify network: `docker network inspect sockshop-chaos-demo_default` |
| Front-end shows blank page | Hard refresh (Ctrl+Shift+R). Edge-router may need 30s to detect front-end. |
| `tc` not found in a container | Only the 4 wrapper-Dockerfile services have `tc`. Use crash tests for Go services. |
| Pipeline fails at Build stage | First build downloads many images (~2GB). Ensure good internet and wait. |
| `CHAOS_CONTAINER` name conflict with voting app | The Sock Shop uses `chaos-controller-sockshop` (different from `chaos-controller-ci`) |

---

## 9. Quick Reference: All URLs During Sock Shop Demo

| Service | URL | Notes |
|---------|-----|-------|
| **Jenkins** | <http://localhost:8080> | Pipeline management |
| **ChaosController Dashboard** | <http://localhost:5050> | Fault injection & experiments |
| **Sock Shop (Web UI)** | <http://localhost:8083> | Full e-commerce storefront |
| **Default Login** | username: `user` / password: `password` | Pre-seeded test account |

---

## 10. File Summary

| File | Location | Purpose |
|------|----------|---------|
| `docker-compose.yml` | `~/sockshop-chaos-demo/` | Modified Sock Shop with chaos capabilities |
| `chaos-config.yml` | `~/sockshop-chaos-demo/` | Defines services, tests, and thresholds |
| `Jenkinsfile` | `~/sockshop-chaos-demo/` | Jenkins pipeline with resilience gate |
| `chaos-dockerfiles/*/Dockerfile` | `~/sockshop-chaos-demo/` | Wrapper Dockerfiles adding iproute2 |

---

*Guide created for ChaosController — Failure Injection Major Project*
*Target App: Weaveworks Sock Shop (https://github.com/microservices-demo/microservices-demo)*
