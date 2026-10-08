# Health Checks and Startup Order

This document describes how each service in the `theepicbook` stack verifies that it is ready to serve traffic and explains the order in which the services start.

The stack consists of four services arranged across two network tiers:

* `reverse-proxy` (Nginx) — public entry point, port 80
* `frontend` — serves static assets, port 80 (internal)
* `backend` — Node/Express application serving SSR pages and the API, port 8080
* `database` — MySQL 8.0, port 3306 (isolated on the internal `back-tier`)

Health checks are used together with `depends_on` conditions to control the startup order. This ensures that a service does not attempt to communicate with a dependency before that dependency is ready.

---

## Health-Check Method per Service

### `database` — MySQL 8.0

* **Command:**
  `mysqladmin ping -h localhost -uroot -p${DB_ROOT_PASSWORD}`

* **Method:** Runs `mysqladmin ping` over the local socket using the root credentials. It returns exit code `0` only when the MySQL server is accepting connections. This is the standard readiness check for MySQL.

* **Timing:** `interval: 10s`, `timeout: 5s`, `retries: 5`, `start_period: 30s`

* **Why these values:** During the first boot, the MySQL entrypoint initializes the data directory, executes the three seed SQL files (`01-schema.sql`, `02-authors.sql`, and `03-books.sql`), and then starts the database server. This initialization can take 30–60 seconds, so a `start_period` of 30 seconds helps prevent false unhealthy states during startup.

### `backend` — Node/Express

* **Command:**
  `wget -qO- http://localhost:8080/health`

* **Method:** Requests the application's `/health` endpoint defined in `backend/server.js` at line 19. A successful HTTP 200 response confirms that the Express server is listening and the application process is healthy. Because `wget` returns a non-zero exit code for an unsuccessful response, the health check automatically fails when the application returns an error.

* **Timing:** `interval: 10s`, `timeout: 5s`, `retries: 5`, `start_period: 20s`

* **Why `wget` instead of `curl`:** The backend image is based on `node:20-alpine`, which includes BusyBox `wget` but does not include `curl` by default. Using `wget` avoids installing an additional package and keeps the image smaller.

### `frontend` — Static Asset Server

* **Command:**
  `wget -q --spider http://localhost/`

* **Method:** Sends a request to the root of the frontend server on port 80. A successful exit indicates that the container is serving files correctly. The `--spider` option is used because only the HTTP response status is required, not the response body.

* **Timing:** `interval: 10s`, `timeout: 5s`, `retries: 5`, `start_period: 10s`

* **Why port 80:** The frontend Dockerfile declares `EXPOSE 80`, and the reverse proxy forwards `/assets/` requests to `frontend:80`.

### `reverse-proxy` — Nginx

* **Command:**
  `wget -qO- http://localhost/health`

* **Method:** Requests `/health` through Nginx, which proxies the request to `backend:8080/health`. A successful HTTP 200 response confirms three things:

  1. Nginx is running and listening on port 80.
  2. The Nginx configuration loaded successfully.
  3. The upstream backend is reachable and responding through the proxy.

* **Timing:** `interval: 10s`, `timeout: 5s`, `retries: 5`, `start_period: 20s`

---

## Startup Dependency Order

Dependencies are declared using `depends_on` together with the `condition: service_healthy` condition. Docker Compose waits for the specified dependency to pass its health check before starting the dependent service.

The resulting startup order is:

```text
database
    │
    │ healthy
    ▼
backend
    │
    │ healthy
    ▼
reverse-proxy

frontend
    │
    │ healthy
    ▼
reverse-proxy
```

This startup strategy improves reliability by ensuring that dependent services only begin operation after their required services have successfully passed their health checks.
