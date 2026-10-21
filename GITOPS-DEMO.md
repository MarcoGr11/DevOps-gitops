# GitOps demo: cheat sheet

Starting state: Git has tag `1.0.0`, `selfHeal: true`; `root` and `booking` are Synced / Healthy.
Outputs below are real outputs from the rehearsal run.

## 0. Preparation (before the defense)

```sh
minikube status                                   # Running
kubectl -n argocd get pods                        # all Running
kubectl -n argocd port-forward svc/argocd-server 8080:443 &        # UI: https://localhost:8080
kubectl -n coworking-booking port-forward svc/booking-api 8081:80 &
argocd login localhost:8080 --username admin --insecure            # asks for the admin password
argocd app list
cd ~/PycharmProjects/DevOps-gitops && git pull
```

Expected:
```
NAME            ...  NAMESPACE          STATUS  HEALTH   SYNCPOLICY  ...  PATH            TARGET
argocd/booking  ...  coworking-booking  Synced  Healthy  Auto-Prune  ...  charts/booking  main
argocd/root     ...  argocd             Synced  Healthy  Auto-Prune  ...  argocd-apps     main
```

## 1. Interface: root -> booking -> tree

1. https://localhost:8080 -> `root` (App of Apps, path `argocd-apps`) -> its only child is the Application `booking`.
2. Open `booking`: Source `charts/booking`, Helm values `values-minikube.yaml`, destination `coworking-booking`.
3. Tree: ConfigMap, PVC, Services `booking-api` / `booking-postgres`, Deployments -> ReplicaSets -> Pods, Ingress.
   No Secret in the tree: it was created by hand (`existingSecret: booking-secret`), not stored in Git.
4. Show that Helm only renders templates:
   ```sh
   helm list -n coworking-booking      # empty
   kubectl -n argocd get application booking -o jsonpath='{.spec.syncPolicy}'; echo
   # {"automated":{"prune":true,"selfHeal":true},"syncOptions":["CreateNamespace=true"]}
   ```
   (`argocd app get` prints only `Automated (Prune)`; selfHeal is visible in the spec above or in the UI.)

## 2. Update via Git: 1.0.0 -> 1.1.0

```sh
curl localhost:8081/api/health/
# {"status":"ok","version":"1.0.0","pod":"booking-api-..."}

sed -i '' 's/tag: "1.0.0"/tag: "1.1.0"/' charts/booking/values-minikube.yaml   # api and worker
git diff
git commit -am "Release booking 1.1.0" && git push
argocd app get booking --refresh        # or the Refresh button in the UI
kubectl -n coworking-booking get pods -w   # Ctrl+C when the new pods are 1/1 and old ones are gone
```

Expected: `OutOfSync from main (<sha>)` right after refresh, then `Synced` / `Healthy`.
Rolling update (maxSurge 1, maxUnavailable 0): a new pod becomes 1/1, then an old one goes Terminating.

```sh
pkill -f "port-forward svc/booking-api"; kubectl -n coworking-booking port-forward svc/booking-api 8081:80 &
curl localhost:8081/api/health/
# {"status":"ok","version":"1.1.0","pod":"booking-api-..."}
```

## 3. Manual change -> selfHeal

```sh
kubectl -n coworking-booking scale deployment booking-api --replicas=5
kubectl -n coworking-booking get deploy booking-api -w
```

Rehearsal output (selfHeal reverted it in about 2 s):
```
18:44:37  booking-api   2/5   5     2
18:44:39  booking-api   2/2   2     2
```

Explanation: without selfHeal (how it was first configured) the app became `OutOfSync` and the 5 replicas
stayed; `selfHeal: true` in `argocd-apps/booking.yaml` was enabled by a commit, `root` applied it.

## 4. Nonexistent tag -> Synced but Degraded -> git revert

```sh
perl -0pi -e 's/(devops-api\n    tag: )"[^"]*"/$1"9.9.9"/' charts/booking/values-minikube.yaml   # api only
git diff
git commit -am "Release booking api 9.9.9" && git push
argocd app get booking --refresh
kubectl -n coworking-booking get pods
```

Expected (Degraded appears after `progressDeadlineSeconds: 60`, i.e. ~1 min; before that Progressing):
```
Sync Status:        Synced to main (<sha>)
Health Status:      Degraded

booking-api-5b84fdf7cc-pnl54   0/1   ImagePullBackOff   0   72s
booking-api-5bcd48fdc6-9g5rv   1/1   Running            0   4m
booking-api-5bcd48fdc6-tgkvj   1/1   Running            0   3m54s
```
Point: Synced = cluster matches Git; Degraded = what is in Git does not work. Old pods still serve:
```sh
curl localhost:8081/api/health/      # still the previous version, status ok
```

Rollback through Git (not `argocd app rollback`: auto-sync would undo it):
```sh
git log --oneline
git revert <sha of "Release booking api 9.9.9"> --no-edit && git push
argocd app get booking --refresh     # OutOfSync -> after ~30 s Synced / Healthy
argocd app history booking           # each sync = a Git commit
curl localhost:8081/api/health/
```

## 5. How to change any value in Git for the teacher

All edits go to `charts/booking/values-minikube.yaml`, then `git commit -am "..." && git push`,
then `argocd app get booking --refresh` (or wait up to ~3 min for the automatic poll).

- Replicas of api:
  ```yaml
  replicaCount:
    api: 3
  ```
  Check: `kubectl -n coworking-booking get deploy booking-api` -> `3/3`.
- Application config (goes to ConfigMap `booking-config`; pods restart because of the `checksum/config` annotation):
  ```yaml
  config:
    bookingTimeoutMinutes: "30"
  ```
  Check: `kubectl -n coworking-booking get cm booking-config -o jsonpath='{.data.BOOKING_TIMEOUT_MINUTES}'` -> `30`.
- Any other key from `charts/booking/values.yaml` (resources, ingress host, ...) is overridden the same way.

To undo: `git revert HEAD --no-edit && git push`.

## 6. If something goes wrong

- `curl` hangs / connection refused after a rollout: port-forward is bound to a deleted pod, restart it:
  `pkill -f "port-forward svc/booking-api"; kubectl -n coworking-booking port-forward svc/booking-api 8081:80 &`
- UI does not open: `pkill -f "port-forward svc/argocd-server"; kubectl -n argocd port-forward svc/argocd-server 8080:443 &`
- `argocd: Unauthenticated` / `invalid session`: `argocd login localhost:8080 --username admin --insecure`
- Change pushed but nothing happens: `argocd app get booking --refresh` (Argo CD polls Git every ~3 min).
  For changes in `argocd-apps/`: `argocd app get root --refresh`.
- App stuck: `argocd app get booking` (MESSAGE column), `kubectl -n coworking-booking get events --sort-by=.lastTimestamp | tail`,
  `kubectl -n coworking-booking describe pod <pod>`. Fix in Git, not by `kubectl apply`.
- `CreateContainerConfigError`: Secret `booking-secret` missing, recreate it (README, "Deploy from scratch", step 2).
- api/worker restart 1-3 times right after a fresh deploy: they start before PostgreSQL is ready, this is normal.
