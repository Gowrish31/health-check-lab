# Cloud Mapping: Local Health Checks to Google Cloud Run & Kubernetes

## 1. Overview & Architecture

In production environments (such as **Google Cloud Run** or **Kubernetes**), application runtimes and load balancers rely on automated HTTP probes to determine container health, readiness, and lifecycle actions.

This repository implements two distinct health check endpoints:
- **`GET /health` (Liveness Check)**: A **shallow** probe that verifies if the Node.js Express web process is running and able to handle HTTP traffic.
- **`GET /ready` (Readiness Check)**: A **deep** probe that executes `SELECT 1` against the PostgreSQL database to verify external dependency connectivity.

---

## 2. Mapping Local Checks to Cloud Run & Kubernetes Probes

| Local Implementation | Cloud Run Equivalent | Kubernetes Equivalent | Purpose & Action on Failure |
| :--- | :--- | :--- | :--- |
| **`GET /health`** | **Liveness Probe** | `livenessProbe` | **Process Survival**: Determines if the app process is alive. On failure, the platform **restarts** the container. It must be shallow (no shared DB dependencies) to prevent cascading restart storms. |
| **`GET /ready`** | **Startup Probe** | `readinessProbe` | **Traffic Routing & Boot Protection**: Determines if the app is ready to serve real user requests. On failure, traffic routing stops (returns 503 / removes from ingress) **without restarting** the container. |

---

## 3. Shallow vs. Deep Probe Philosophy & The DB Trap

### Why `GET /health` Must Be Shallow
If a liveness check pinged a shared database during a temporary 30-second DB outage, **every application instance in the fleet would fail liveness simultaneously**. The cloud runtime (Cloud Run or Kubernetes) would kill and restart all instances at once. This creates a **cascading outage**: cold-starting app instances hammer an already struggling database upon boot.

### Why `GET /ready` Must Be Deep
A readiness check must verify database connectivity. If PostgreSQL goes down:
1. `GET /ready` returns `HTTP 503 Service Unavailable`.
2. The load balancer / ingress immediately stops sending user requests to this instance.
3. The container stays running cleanly without dying.
4. When PostgreSQL recovers, `GET /ready` returns `HTTP 200 OK`, and traffic automatically resumes.

---

## 4. Empirical Validation Evidence

### Phase 1: Healthy State
Both services (`db` and `orders-api`) running normally.

```bash
$ docker compose ps
NAME                            IMAGE                         COMMAND                  SERVICE      CREATED         STATUS                  PORTS
health-check-lab-db-1           postgres:16                   "docker-entrypoint.s…"   db           3 minutes ago   Up 3 minutes            0.0.0.0:5432->5432/tcp
health-check-lab-orders-api-1   health-check-lab-orders-api   "docker-entrypoint.s…"   orders-api   3 minutes ago   Up 3 minutes (healthy)  0.0.0.0:3000->3000/tcp

$ curl -i http://localhost:3000/health
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 2

OK

$ curl -i http://localhost:3000/ready
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 5

READY
```

### Phase 2: Dependency Failure (Database Stopped)
The PostgreSQL database is stopped using `docker compose stop db`.

```bash
$ docker compose stop db
Container health-check-lab-db-1 Stopped

$ curl -i http://localhost:3000/health
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 2

OK

$ curl -i http://localhost:3000/ready
HTTP/1.1 503 Service Unavailable
Content-Type: text/html; charset=utf-8
Content-Length: 9

NOT READY
```

> **Observation**: `/health` remains 200 OK (process alive), while `/ready` accurately reports 503 Service Unavailable (dependency down).

### Phase 3: Recovery State (Database Restored)
The PostgreSQL database is restarted using `docker compose start db`.

```bash
$ docker compose start db
Container health-check-lab-db-1 Started

$ curl -i http://localhost:3000/ready
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 5

READY

$ curl -i http://localhost:3000/orders
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

[{"id":3,"customer_name":"Charlie Brown","total_amount":"99.99","status":"Delivered", ...}]
```

---

## 5. Cloud Run YAML Specification Example (Reference Only)

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: orders-api
spec:
  template:
    spec:
      containers:
      - image: gcr.io/my-project/orders-api:latest
        ports:
        - containerPort: 3000
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 20
          periodSeconds: 10
        startupProbe:
          httpGet:
            path: /ready
            port: 3000
          failureThreshold: 3
          periodSeconds: 10
```
