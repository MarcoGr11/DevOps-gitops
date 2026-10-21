# DevOps-gitops

GitOps configuration for the coworking booking system (Django REST API, worker, PostgreSQL),
deployed to minikube by Argo CD. Git is the single source of truth: no manual `helm install` / `kubectl apply`
of the chart.

## Structure

```
bootstrap/root.yaml          # App of Apps "root" -> watches argocd-apps/
argocd-apps/booking.yaml     # Application "booking" -> charts/booking with values-minikube.yaml
charts/booking/              # Helm chart (Chart.yaml, values*.yaml, templates/)
  values-minikube.yaml       # what Argo CD deploys: image tags, replicas, ingress, flags
```

`root` (namespace `argocd`) creates `booking`; `booking` renders the chart with Helm and syncs it
into namespace `coworking-booking` (created by `CreateNamespace=true`). Helm is used only for
templating, so `helm list` there is empty.

## Deploy from scratch

1. Install Argo CD:
   ```sh
   kubectl create namespace argocd
   kubectl apply -n argocd --server-side --force-conflicts \
     -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```
2. Create the application Secret (outside Git):
   ```sh
   kubectl create namespace coworking-booking
   kubectl -n coworking-booking create secret generic booking-secret \
     --from-literal=POSTGRES_PASSWORD="$(openssl rand -hex 16)" \
     --from-literal=DJANGO_SECRET_KEY="$(openssl rand -hex 32)"
   ```
3. Bootstrap:
   ```sh
   kubectl apply -f bootstrap/root.yaml
   ```
4. Check:
   ```sh
   argocd app list
   kubectl -n coworking-booking port-forward svc/booking-api 8081:80
   curl localhost:8081/api/health/
   ```

Every further change (image tag, replicas, config) is a commit to `main`; Argo CD syncs it automatically.

## existingSecret

Credentials never go to Git. With `existingSecret: booking-secret` the chart does not render its own
Secret (and does not require `secret.*` values); api, worker and postgres read `POSTGRES_PASSWORD` and
`DJANGO_SECRET_KEY` from the pre-created Secret. With `existingSecret: ""` (default) the chart behaves as
a plain Helm chart and renders the Secret from `secret.*`.

## Argo CD specifics in values-minikube.yaml

- `hooks.dbCheck.enabled: false`: Argo CD turns Helm pre-install/pre-upgrade hooks into PreSync and runs
  them on every sync, including the first one when PostgreSQL does not exist yet.
- `tests.enabled: false`: no `helm test` pod in the application tree.
- `progressDeadlineSeconds: 60`: a stuck rollout (e.g. nonexistent image tag) turns Degraded after 1 min.
