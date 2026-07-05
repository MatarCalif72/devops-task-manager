# Changes I made to the original app

This file explains every change made to the original forked app.

## 1. Changed a port number for local testing (docker-compose.yml)

My computer already had another Postgres database using port 5432, so
Docker couldn't also use that port. I changed the local port mapping to
5433 instead. This only affects testing on my own machine — the containers
still talk to each other normally, and this doesn't affect Kubernetes at all.

## 2. Fixed a crash bug in the backend (backend/server.js)

While testing the requirement "if the database connection breaks, the
backend pod should become not ready," I found that the app didn't just
become "not ready" — it actually crashed and kept restarting in a loop.

The reason: the database library the app uses will crash the whole program
if the connection drops and nobody is "listening" for that kind of error. I
added a few lines so the app catches that error instead of crashing. Now,
if the database goes down, the backend keeps running and simply reports
"unhealthy" until the database comes back — exactly what the assignment
asked for.