# Лабораторна №5 — чекліст скриншотів

10 скриншотів, пункти 3–6 методички (пункти 1–2 закриті: образи
`ghcr.io/marcogr11/devops-api|worker:1.0.0|1.1.0` опубліковані, версія видна в
`/api/health/`). Кожен PNG кладеться в `reports/screens/лаб5/` під іменем із
заголовка `###`. Один блок команд = один скриншот (якщо не влазить в одне вікно,
кілька знімків одного блоку склеюються вертикально). `reports/build_report.py`
бере звідси «Що робимо», «Підпис» і «Що видно» для PDF. «Що видно» заповнюється
після перегляду справжніх скриншотів.

Вікно терміналу — не ширше ~1800 px (приблизно 130 символів). Усі команди
виконуються з кореня репозиторію `DevOps-gitops`.

**Як копіювати:** кожен блок коду вставляється в термінал **цілком**, одним
копіюванням. Рядки, що закінчуються на `\`, — це одна команда, перенесена для
читабельності. Коментарів `#` у блоках немає (zsh без `setopt interactivecomments`
відповідає на них `command not found: #`).

## Порядок виконання

Розділи в PDF ідуть за методичкою, знімати треба в такому порядку:

`підготовка → 5-01 → 5-02 → 5-04 → 5-03 → 5-05a → 5-05b → (прибрати selfHeal) → 5-06a → (повернути selfHeal) → 5-06b → 5-06c → 5-06d → після зйомки`

**5-04 (CLI) знімається перед 5-03 (UI)**, щоб у дереві UI було видно ту саму
ревізію і поди, що в терміналі. **Перед 5-06a selfHeal тимчасово прибирається**,
після 5-06a — повертається (див. блоки між 5-05b і 5-06b).

## Підготовка (не знімається)

**Термінал №2** (лишити працювати; UI Argo CD — https://localhost:8080):

```bash
pkill -f "port-forward svc/argocd-server"
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

**Термінал №3** (застосунок на 8081; після кожного rollout перезапускати цим самим блоком):

```bash
pkill -f "port-forward svc/booking-api"
kubectl -n coworking-booking port-forward svc/booking-api 8081:80
```

**Термінал №1** (основний, з нього всі скриншоти):

```bash
cd ~/PycharmProjects/DevOps-gitops
git pull
argocd login localhost:8080 --username admin --insecure
```

Пароль вводиться вручну на запит і в кадр не потрапляє. Після логіну очистити
екран (`clear` або Cmd+K).

Стартовий стан: у Git тег `1.0.0`, `selfHeal: true`, `argocd app list` —
`root` і `booking` Synced / Healthy.

Паролі в кадр не потрапляють:
- не запускати `argocd login` / `argocd account update-password` у кадрі;
- не запускати `kubectl get secret ... -o yaml`, `kubectl describe secret`
  з `-o`, `env`, `printenv`, `kubectl exec ... env`, `helm get values`;
- у UI не відкривати вкладку Secret (її в дереві і нема: `existingSecret`)
  і сторінку Settings → Accounts.

---

## 3. Репозиторій конфігурації і встановлення Argo CD

### 5-01-repo.png — 3.a GitOps-репозиторій

**Що робимо:** Показуємо структуру репозиторію конфігурації, його видимість і Application `booking`, що вказує на чарт.

**Команди:**
```bash
gh repo view MarcoGr11/DevOps-gitops --json url,visibility
find . -path ./.git -prune -o -path ./reports -prune -o -type f -print | sort
cat argocd-apps/booking.yaml
```

**Має бути видно:** `"visibility":"PUBLIC"`; файли `bootstrap/root.yaml`, `argocd-apps/booking.yaml`, `charts/booking/Chart.yaml`, `values*.yaml` (зокрема `values-minikube.yaml`), `templates/…`, `README.md`, `GITOPS-DEMO.md`; у `booking.yaml` — `repoURL`, `path: charts/booking`, `valueFiles: values-minikube.yaml`, `namespace: coworking-booking`, `automated: prune/selfHeal`, `CreateNamespace=true`.

**Не має бути:** токена `gh` (не запускати `gh auth status --show-token`).

**Підпис:** Структура репозиторію DevOps-gitops і Application booking

**Що видно:**

### 5-02-argocd-install.png — 3.b Argo CD встановлено

**Що робимо:** Перевіряємо поди Argo CD, версії клієнта і сервера та те, що початковий пароль видалено.

**Команди:**
```bash
kubectl -n argocd get pods
argocd version --short
kubectl -n argocd get secret argocd-initial-admin-secret
```

**Має бути видно:** 7 подів `argocd-*` у `Running` (application-controller, applicationset-controller, dex-server, notifications-controller, redis, repo-server, server); `argocd: v3.5.6…` і `argocd-server: v3.5.6…`; `NotFound` для `argocd-initial-admin-secret`.

**Не має бути:** виводу `argocd admin initial-password`, команди `argocd login` з паролем.

**Підпис:** Поди Argo CD, версії клієнта й сервера, видалений initial-admin-secret

**Що видно:**

---

## 4. Застосунок під керуванням Argo CD

### 5-03-ui-tree.png — 4.a Інтерфейс: root → booking → дерево

**Що робимо:** Відкриваємо UI https://localhost:8080 → Applications → `root` (бачимо дочірній Application `booking`) → клік на `booking` → дерево ресурсів.

**Команди:** — (знімок вікна браузера: сторінка застосунку `booking`, вид Tree).

**Має бути видно:** статуси `Healthy` і `Synced` (до `main`, той самий коміт, що в 5-04); дерево: `booking-config`, `booking-postgres-data`, Services `booking-api`/`booking-postgres`, Deployments → ReplicaSets → Pods (2 api, 1 worker, 1 postgres), Ingress `booking-api`. Якщо влазить — рядок адреси `localhost:8080/applications/argocd/booking`.

**Не має бути:** відкритої панелі з паролем/токеном, сторінки Settings → Accounts.

**Підпис:** Дерево ресурсів застосунку booking в інтерфейсі Argo CD

**Що видно:**

### 5-04-bootstrap.png — 4.b Bootstrap через root (App of Apps)

**Що робимо:** Повторно застосовуємо єдиний ручний маніфест `bootstrap/root.yaml` і перевіряємо, що `root` створив `booking`, а Helm лише рендерить шаблони.

**Команди:**
```bash
kubectl apply -f bootstrap/root.yaml
argocd app list
argocd app get booking | grep -E "Path|Helm Values|Sync Policy|Sync Status|Health Status"
kubectl -n coworking-booking get pods
helm list -n coworking-booking
curl -s localhost:8081/api/health/; echo
```

**Має бути видно:** `application.argoproj.io/root unchanged`; `argocd/root` і `argocd/booking` — `Synced` `Healthy`; `Path: charts/booking`, `Helm Values: values-minikube.yaml`; 4 поди `Running`; порожній `helm list` (лише заголовок); `"version":"1.0.0"`.

**Не має бути:** `--set secret.*`, виводу Secret.

**Підпис:** root і booking у стані Synced/Healthy, порожній helm list, версія 1.0.0

**Що видно:**

---

## 5. Оновлення через Git

### 5-05a-release-commit.png — 5.a Зміна тегу і push

**Що робимо:** Змінюємо тег образів api і worker на `1.1.0` у `values-minikube.yaml` і пушимо коміт у `main`.

**Команди:**
```bash
sed -i '' 's/tag: "1.0.0"/tag: "1.1.0"/' charts/booking/values-minikube.yaml
git diff
git commit -am "Release booking 1.1.0"
git push
git log --oneline -3
```

**Має бути видно:** у `git diff` два рядки `-    tag: "1.0.0"` / `+    tag: "1.1.0"`; коміт і `main -> main` у виводі push.

**Не має бути:** токена в URL remote.

**Підпис:** Оновлення тегу до 1.1.0 комітом у репозиторій конфігурації

**Що видно:**

### 5-05b-release-result.png — 5.b Синхронізація і нова версія

**Що робимо:** Просимо Argo CD перечитати Git, чекаємо Synced/Healthy, дивимось поди й перевіряємо версію. Перед `curl` перезапустити port-forward у терміналі №3 (старий тримається за видалений под).

**Команди:**
```bash
argocd app get booking --refresh | grep -E "Sync Status|Health Status"
argocd app wait booking --sync --health --timeout 180 | grep -E "Sync Status|Health Status"
kubectl -n coworking-booking get pods
argocd app history booking
```

Тепер перезапустити термінал №3 і в терміналі №1 (у тому ж кадрі):

```bash
curl -s localhost:8081/api/health/; echo
```

**Має бути видно:** одразу після refresh `OutOfSync from main (<коміт з 5-05a>)`, потім `Synced` / `Healthy`; нові поди api з віком у секундах (старі можуть бути `Terminating`); в історії нова ревізія з тим самим комітом; `"version":"1.1.0"`.

**Підпис:** Argo CD синхронізував коміт, rolling update до 1.1.0

**Що видно:**

---

## 6. Дрейф, selfHeal, Degraded і відкат

### Прибрати selfHeal (не знімається, перед 5-06a)

```bash
sed -i '' '/selfHeal: true/d' argocd-apps/booking.yaml
git diff
git commit -am "Disable selfHeal for drift demo"
git push
argocd app get root --refresh | grep -E "Sync Status|Health Status"
argocd app wait root --sync --health --timeout 120 | grep -E "Sync Status|Health Status"
kubectl -n argocd get application booking -o jsonpath='{.spec.syncPolicy}'; echo
argocd app wait booking --sync --health --timeout 120 | grep -E "Sync Status|Health Status"
```

Останні два рядки: `{"automated":{"prune":true},…}` (без `selfHeal`) і booking `Synced` `Healthy`. Лише тоді знімати 5-06a.
У `git diff` має зникнути тільки рядок `selfHeal: true` у `argocd-apps/booking.yaml` (у `bootstrap/root.yaml` він лишається).

### 5-06a-drift.png — 6.a Ручна зміна без selfHeal

**Що робимо:** Масштабуємо `booking-api` вручну до 5 реплік, коли selfHeal вимкнено, і дивимось, що бачить Argo CD.

**Команди:**
```bash
kubectl -n argocd get application booking -o jsonpath='{.spec.syncPolicy}'; echo
kubectl -n coworking-booking scale deployment booking-api --replicas=5
sleep 30
argocd app get booking --refresh | grep -E "Sync Status|Health Status|Deployment .*booking-api"
kubectl -n coworking-booking get deploy booking-api
```

**Має бути видно:** `syncPolicy` без `selfHeal`; через 30 с `Sync Status: OutOfSync`, рядок Deployment `booking-api` — `OutOfSync`; `booking-api 5/5` — ручна зміна лишилась.

**Підпис:** Дрейф без selfHeal: OutOfSync, 5 реплік лишаються

**Що видно:**

### Повернути selfHeal (не знімається, одразу після 5-06a)

```bash
git revert HEAD --no-edit
git push
argocd app get root --refresh | grep -E "Sync Status|Health Status"
argocd app wait root --sync --health --timeout 120 | grep -E "Sync Status|Health Status"
kubectl -n argocd get application booking -o jsonpath='{.spec.syncPolicy}'; echo
argocd app wait booking --sync --health --timeout 120 | grep -E "Sync Status|Health Status"
kubectl -n coworking-booking get deploy booking-api
```

`HEAD` тут — коміт «Disable selfHeal for drift demo» (між ним і revert нічого не комітити).
Очікування: root `Synced` `Healthy`; `syncPolicy` знову з `"selfHeal":true`; booking `Synced` `Healthy`; `booking-api 2/2` (новий коміт → Argo CD синхронізує booking і повертає 2 репліки).

### 5-06b-selfheal.png — 6.b selfHeal повертає стан з Git

**Що робимо:** Повторюємо ручне масштабування з увімкненим selfHeal і кілька разів дивимось на Deployment.

**Команди:**
```bash
kubectl -n argocd get application booking -o jsonpath='{.spec.syncPolicy}'; echo
kubectl -n coworking-booking scale deployment booking-api --replicas=5
for i in 1 2 3 4 5; do date +%T; kubectl -n coworking-booking get deploy booking-api --no-headers; sleep 2; done
argocd app get booking | grep -E "Sync Status|Health Status"
```

**Має бути видно:** `"selfHeal":true`; у перших рядках `2/5` (або `5/5`), через кілька секунд `2/2`; `Synced` / `Healthy`.

**Підпис:** selfHeal: Argo CD повертає 2 репліки з Git за кілька секунд

**Що видно:**

### 5-06c-degraded.png — 6.c Synced, але Degraded

**Що робимо:** Ставимо неіснуючий тег `9.9.9` лише для api, пушимо і чекаємо, поки сплине `progressDeadlineSeconds: 60`.

**Команди:**
```bash
perl -0pi -e 's/(devops-api\n    tag: )"[^"]*"/$1"9.9.9"/' charts/booking/values-minikube.yaml
git diff
git commit -am "Release booking api 9.9.9"
git push
argocd app get booking --refresh | grep -E "Sync Status|Health Status"
argocd app wait booking --degraded --timeout 180 | grep -E "Sync Status|Health Status"
kubectl -n coworking-booking get pods
kubectl -n coworking-booking get deploy booking-api \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status} {.reason}{"\n"}{end}'
curl -s localhost:8081/api/health/; echo
```

**Має бути видно:** `"1.1.0"` → `"9.9.9"` у diff (лише api); через ~1 хв `Sync Status: Synced`, `Health Status: Degraded`; новий под api у `ImagePullBackOff` (або `ErrImagePull`), два старі `Running`; `Progressing=False ProgressDeadlineExceeded`; `curl` відповідає `"status":"ok","version":"1.1.0"`.

**Підпис:** Synced, але Degraded: неіснуючий тег 9.9.9, старі поди працюють

**Що видно:**

### 5-06d-revert.png — 6.d Відкат через git revert

**Що робимо:** Відкочуємо коміт з `9.9.9` через `git revert` (не `argocd app rollback`, бо ввімкнена автосинхронізація) і перевіряємо результат.

**Команди:**
```bash
git log --oneline -4
git revert $(git log --grep="9.9.9" --format=%h -1) --no-edit
git push
argocd app get booking --refresh | grep -E "Sync Status|Health Status"
argocd app wait booking --sync --health --timeout 180 | grep -E "Sync Status|Health Status"
argocd app history booking
kubectl -n coworking-booking get pods
curl -s localhost:8081/api/health/; echo
```

**Має бути видно:** коміт `Revert "Release booking api 9.9.9"`; після refresh `OutOfSync … Degraded`, потім `Synced` / `Healthy`; в історії останні ревізії — коміт 9.9.9 і коміт revert; лише 4 поди `Running`, без `ImagePullBackOff`; `"version":"1.1.0"`.

**Підпис:** Відкат через git revert: знову Synced/Healthy, версія 1.1.0

**Що видно:**

---

## Після зйомки (не знімається)

Повернути тег `1.0.0`, щоб на захисті наживо показати оновлення до 1.1.0:

```bash
sed -i '' 's/tag: "1.1.0"/tag: "1.0.0"/' charts/booking/values-minikube.yaml
git commit -am "Reset booking to 1.0.0 for live demo"
git push
argocd app get booking --refresh | grep -E "Sync Status|Health Status"
argocd app wait booking --sync --health --timeout 180 | grep -E "Sync Status|Health Status"
argocd app list
git status
```

Очікування: обидва застосунки `Synced` `Healthy`, `git status` чистий. Потім перезапустити термінал №3
і `curl -s localhost:8081/api/health/` → `"version":"1.0.0"`.
