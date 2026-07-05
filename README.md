# DevOps Task Manager

forked from [yehudits/devops-task-manager](https://github.com/yehudits/devops-task-manager) —
containerized, deployed to AWS EKS via Kubernetes, and built/deployed through a
Jenkins CI/CD pipeline triggered by GitHub webhooks.

See [CHANGES.md](./CHANGES.md) for a full list of deviations from the original
source code and instructions.

## CI/CD pipeline

On every push to `main`, a GitHub webhook triggers a Jenkins pipeline that:

1. Builds the backend and frontend Docker images, tagged with the Jenkins
   build number (`vN`) — never overwriting previous versions.
2. Pushes both images to their AWS ECR repositories.
3. Updates the `backend` and `frontend` Deployments on EKS to the new image
   tags and waits for a zero-downtime rollout to complete.

## Running locally

```
docker compose up --build
```

App is available at `http://localhost:3000`.

## Screenshots

**App running and reachable from the browser, via Ingress:**

![App via ingress](./screenshots/app-via-ingress.png)

**`kubectl get pods,svc,ingress` — backend and postgres have no external IP,
confirming only the frontend is exposed:**

![kubectl output](./screenshots/kubectl-get-pods-svc-ingress.png)

**Jenkins pipeline, build #1 (manually triggered, green):**

![Jenkins build 1](./screenshots/jenkins-build-1.png)

**Jenkins pipeline, build #2 (automatically triggered by a GitHub push via
webhook, green):**

![Jenkins build 2](./screenshots/jenkins-build-2-webhook-triggered.png)

**Jenkins build history overview:**

![Jenkins build history](./screenshots/jenkins-build-history.png)
