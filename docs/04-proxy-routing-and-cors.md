
# Proxy Routing and CORS

This document describes how the reverse proxy (`reverse-proxy`, `nginx:alpine`) routes traffic to the frontend and backend containers and explains whether CORS is required for the application to function.

---

## Proxy Architecture

All external traffic enters the stack through a single public port: **80**, served by the `reverse-proxy` service.

The proxy inspects the request path and forwards the traffic to one of two upstream services:

- `backend:8080` — Node/Express application providing SSR pages, JSON API endpoints, and the health endpoint.
- `frontend:80` — Static file server serving CSS, JavaScript, images, and other frontend assets.

The upstream services are resolved using Docker's embedded DNS on the `front-tier` network. This allows the Nginx configuration to use the Docker service names (`backend` and `frontend`) directly instead of relying on container IP addresses.

---

## Route Table

| Location | Upstream | Purpose | Notes |
|---|---|---|---|
| `location = /health` | `http://backend:8080/health` | Health probe | Exact match (`=`) bypasses the other routing rules so monitoring can always reach the health endpoint. |
| `location /api/` | `http://backend:8080` | JSON API | Adds CORS headers and handles `OPTIONS` preflight requests with a `204` response. |
| `location /assets/` | `http://frontend:80` | Static files | CSS, JavaScript, images, and other static assets are served by the frontend container. |
| `location /` | `http://backend:8080` | Application pages | Catch-all route that forwards application requests to the backend for server-rendered HTML. |

Every proxied request also sets standard forwarding headers so that the upstream application can identify the original client request.

```
