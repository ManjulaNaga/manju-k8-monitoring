# Jenkins + Prometheus + Grafana

## Layout
```
jenkins-setup/
├── docker-compose.yml          # jenkins, blackbox, cadvisor, prometheus, grafana
├── Dockerfile / plugins.txt    # Jenkins image with JCasC, Job DSL, Prometheus plugin, docker CLI
├── jenkins.yaml                # JCasC: admin user, git creds, sample-job
├── .env.example                # copy to .env
├── blackbox/blackbox.yml       # probe modules (tcp_connect, http_2xx)
├── prometheus/prometheus.yml   # scrape jobs + liveness/readiness/startup probes
├── prometheus/alerts.yml       # alert rules
└── grafana/
    ├── provisioning/           # datasource + dashboard provider (auto-loaded)
    └── dashboards/jenkins-health.json
```

## Run
```bash
cp .env.example .env      # set passwords and token
docker compose up -d --build
docker compose ps         # jenkins shows (health: starting) -> (healthy)
```

| Service    | URL                              |
|------------|----------------------------------|
| Jenkins    | http://localhost:8080            |
| Grafana    | http://localhost:3000  (dashboard "Jenkins Health" is the home page) |
| Prometheus | http://localhost:9090  (Status → Targets, Alerts) |
| cAdvisor   | http://localhost:8081            |
| Blackbox   | http://localhost:9115            |

## How the probes map

| Probe     | Docker (restart/health)                       | Prometheus (dashboard + alerts)                       |
|-----------|-----------------------------------------------|-------------------------------------------------------|
| Startup   | `healthcheck.start_period: 180s` grace         | HTTP 2xx on `/prometheus/` (only after plugins load); alert if not up 5 min after container start |
| Liveness  | `curl /login` every 30s, unhealthy after 5 misses | TCP connect to `:8080` and `:50000`                 |
| Readiness | —                                             | HTTP 2xx on `/login` (Jenkins returns 503 while loading) |

Docker marks a container *unhealthy* but does not restart it. To auto-restart
on failed liveness, add an autoheal container or run on Kubernetes.

## Metrics
* CPU / memory / network / throttling per container — cAdvisor (`container_*`)
* % of limit — uses the `deploy.resources.limits` on the jenkins service (2 CPU, 2 GB)
* JVM heap, executors, queue — Jenkins Prometheus plugin at `/prometheus/`

## Troubleshooting
* **Dashboard panels empty** — check Prometheus → Status → Targets; all should be UP.
* **Jenkins-internals panels empty** — open http://localhost:8080/prometheus/ and
  confirm metric names (they vary slightly between plugin versions).
* **cAdvisor on Docker Desktop (Mac)** — it sees containers through the Docker VM.
  If container panels are empty, run `docker compose logs cadvisor`.
* **Reload Prometheus config** without restart: `curl -X POST localhost:9090/-/reload`
