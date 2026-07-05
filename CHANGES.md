# Changes from the original forked repo

This file tracks deviations made from the original `yehudits/devops-task-manager`
source during local setup and the DevOps assignment work, with the reasoning behind
each one.

## docker-compose.yml

- **`postgres-db` host port remapped from `5432:5432` to `5433:5432`.**
  Reason: the local dev machine had native Windows PostgreSQL services
  (`postgresql-x64-16` / `postgresql-x64-17`) already bound to port 5432.
  This is purely a host-side port mapping for local `docker compose up` testing —
  containers still talk to each other over the internal Docker network using the
  service name `postgres-db` on its container port `5432`, so the app's behavior
  is unaffected. Does not affect Kubernetes deployment (no equivalent host port
  mapping exists there).

- **`frontend` service's `BACKEND_URL` changed from `http://localhost:3001`
  (browser-facing) to `http://backend:3001`** (Docker network service name).

## backend/server.js

- **Added a `pool.on('error', ...)` handler to the pg `Pool`.**
  Reason: discovered while testing the "backend pod becomes not ready on DB
  connectivity loss" requirement on minikube. Without this handler, when an
  already-established DB connection drops (e.g. Postgres pod deleted/restarted
  while backend is running), node-postgres's `Pool` emits an unhandled
  `'error'` event, which crashes the Node process with an uncaught exception —
  causing the pod to restart (CrashLoopBackOff) instead of staying alive and
  reporting unhealthy via `/health`. This handler just logs the error and lets
  the pool recover/reconnect, so the readiness probe (not a crash/restart) is
  what reflects DB connectivity issues, matching the assignment's requirement.

## frontend/server.js and frontend/package.json

- **Added a server-side reverse proxy for `/api/*`, forwarding to the backend.**
  Reason: the assignment requires exposing **only the frontend** via the
  Kubernetes Ingress — the backend must not be reachable from outside the
  cluster. But the original frontend code fetches `${backendUrl}/api/tasks`
  **directly from the browser** (see `frontend/public/index.html`), using
  whatever URL the `/config` endpoint hands it. If the backend isn't publicly
  exposed, the browser can never reach it directly.

  Fix: `frontend/server.js` now uses `http-proxy-middleware` to proxy any
  `/api/*` request server-side to the backend (via `BACKEND_URL`, now an
  internal-only address — e.g. the Kubernetes Service DNS name
  `http://backend:3001`). The `/config` endpoint now returns an empty
  `backendUrl`, so the existing browser JS's `fetch(`${backendUrl}/api/tasks`)`
  becomes a same-origin relative call (`/api/tasks`) against the frontend
  itself, which the frontend then forwards internally. No changes were needed
  in `public/index.html`.

  Net effect: the browser only ever talks to the frontend. The backend
  Kubernetes Service stays ClusterIP-only with zero external exposure,
  satisfying the "expose frontend only via Ingress" requirement while keeping
  the app fully functional.

## k8s/backend-deployment.yaml and k8s/frontend-deployment.yaml

- **Added a `preStop` lifecycle hook (`sleep 5`) to both containers.**
  Reason: found via load-testing a rolling update on minikube — with just
  `maxUnavailable: 0` / `maxSurge: 1`, ~2% of in-flight requests during a
  rollout got a `504` anyway. This is a well-known Kubernetes race: a pod is
  marked `Terminating` and sent `SIGTERM` immediately, but it takes a moment
  for the Service/Ingress endpoints to stop routing new traffic to it. The
  `preStop` hook delays actual shutdown by 5s, giving that propagation time to
  finish before the container exits, so no requests get routed to a pod that's
  already gone. Re-tested after adding this: 0 failed requests during a full
  rolling update.
