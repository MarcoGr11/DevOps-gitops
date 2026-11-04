# Лабораторна №6 — чекліст скриншотів

12 скриншотів, пункти 1–5 методички. Кожен PNG кладеться в `reports/screens/лаб6/`
під іменем із заголовка `###`. Один блок команд = один скриншот (якщо не влазить в
одне вікно, кілька знімків одного блоку склеюються вертикально).
`reports/build_report.py` бере звідси «Що робимо», «Підпис» і «Що видно» для PDF.
«Що видно» заповнюється після перегляду справжніх скриншотів, поки це поле порожнє.

Вікно терміналу — не ширше ~1800 px (приблизно 130 символів). Кожен блок вставляється
**цілком**, одним копіюванням. Коментарів `#` у блоках немає.

## Порядок зйомки

`підготовка → 6-01 → 6-02 → 6-03 → 6-04 → 6-05 → 6-06 → 6-07 → 6-08 → 6-09 → 6-10 → 6-11 → 6-12 → після зйомки`

- **6-01..6-03** — статичний стан, їх можна знімати повторно.
- **6-04..6-06, 6-08..6-12** — знімаються під час **живого проходу** сценарію. Кожен блок змінює кластер,
  тому порядок міняти не можна. Перезняти один скрін окремо не вийде: треба повернутись у підготовку й
  пройти сценарій спочатку.
- **6-07** знімається, поки реліз стоїть на ручній паузі: пауза не закінчиться сама, час є.
- **6-11** знімається до 6-12: стан Degraded триває, доки не зроблено revert.

## Підготовка перед зйомкою (не знімається)

**Стан `main` до початку:** `image.api.tag: "1.1.0"`, `replicaCount.api: 5`. Починаємо з тегу **1.1.0**.

**Термінал №2** (UI Argo CD, https://localhost:8080):

```bash
pkill -f "port-forward svc/argocd-server"
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

**Термінал №3** (застосунок на 8081):

```bash
pkill -f "port-forward svc/booking-api"
kubectl -n coworking-booking port-forward svc/booking-api 8081:80
```

**Термінал №1** (основний, з нього всі скриншоти):

```bash
cd ~/PycharmProjects/DevOps-gitops
git pull
argocd login localhost:8080 --username admin --insecure
grep -n 'tag:' charts/booking/values-minikube.yaml
argocd app list
kubectl argo rollouts get rollout booking-api -n coworking-booking
```

Пароль вводиться вручну на запит і в кадр не потрапляє. Потім очистити екран (Cmd+K).

Очікування: рядок 8 — `tag: "1.1.0"`; `root`, `booking`, `argo-rollouts` — `Synced  Healthy`; Rollout `Healthy`,
`1.1.0 (stable)`, 5 подів. Якщо тег `1.2.0` або Rollout не Healthy, спершу виконайте блок «Після зйомки».

Паролі в кадр не потрапляють:
- не запускати `argocd login` у кадрі;
- не запускати `kubectl get secret ... -o yaml`, `env`, `printenv`, `kubectl exec ... env`, `helm get values`;
- у браузері не відкривати Settings → Accounts і збережені паролі;
- вимкнути сповіщення IDE і macOS (режим «Не турбувати»).

---

## 1. Контролер Argo Rollouts через Argo CD

### 6-01-controller.png — 1. Контролер і плагін встановлено

**Що робимо:** Перевіряємо, що Application `argo-rollouts` з Git розгорнув контролер і CRD, і що встановлено плагін `kubectl argo rollouts`.

**Перед скріном:** `argo-rollouts` Synced / Healthy.

**Команди:**
```bash
cd ~/PycharmProjects/DevOps-gitops
argocd app get argo-rollouts | grep -E "Sync Status|Health Status"
kubectl -n argo-rollouts get pods
kubectl get crd | grep argoproj
kubectl argo rollouts version
```

**Має бути видно:** `Synced to 2.43.8`, `Healthy`; 2 поди `argo-rollouts-*` у `Running`; CRD `rollouts.argoproj.io` і `analysistemplates.argoproj.io`; `kubectl-argo-rollouts: v1.10.0`.

**Не має бути видно:** паролів admin Argo CD, токенів, сповіщень IDE.

**Підпис:** Рис. 1. Контролер Argo Rollouts, його CRD і плагін kubectl

**Що видно:**

---

## 2. Rollout у Helm-чарті

### 6-02-chart.png — 2. Шаблон Rollout замість Deployment

**Що робимо:** Показуємо шаблони чарту і те, що з `values-minikube.yaml` рендериться Rollout, а не Deployment для api.

**Перед скріном:** будь-який стан, команди лише читають файли.

**Команди:**
```bash
cd ~/PycharmProjects/DevOps-gitops
find charts/booking/templates -name "*rollout*" -o -name "*deployment-api*"
helm template booking charts/booking -f charts/booking/values-minikube.yaml | grep -E "^kind:" | sort | uniq -c
```

**Має бути видно:** файли `rollout-api.yaml` і `deployment-api.yaml`; `1 kind: Rollout`, `2 kind: Deployment` (worker і postgres), без Secret, Job і Namespace.

**Не має бути видно:** паролів admin Argo CD, токенів, сповіщень IDE.

**Підпис:** Рис. 2. Шаблони Rollout і Deployment, перелік об'єктів після рендеру

**Що видно:**

---

## 3. Стратегія розгортання

### 6-03-strategy.png — 3. Canary у values і Rollout на базовій версії

**Що робимо:** Показуємо кількість реплік і кроки canary у values, потім Rollout на стабільній версії.

**Перед скріном:** тег `1.1.0`, Rollout Healthy.

**Команди:**
```bash
cd ~/PycharmProjects/DevOps-gitops
sed -n '/^replicaCount:/,/^  worker:/p;/^rollout:/,$p' charts/booking/values-minikube.yaml
kubectl argo rollouts get rollout booking-api -n coworking-booking
```

**Має бути видно:** `api: 5`; три `setWeight` (20, 60, 80), `pause: {}` і дві паузи `duration: 30s`; `Status: ✔ Healthy`, `Strategy: Canary`, `Step: 6/6`, `1.1.0 (stable)`, 5 подів `Running`.

**Не має бути видно:** паролів admin Argo CD, токенів, сповіщень IDE.

**Підпис:** Рис. 3. Кроки canary у values-minikube.yaml і Rollout у стані Healthy на версії 1.1.0

**Що видно:**

---

## 4. Реліз через Git

Далі живий прохід: виконуйте блоки один за одним.

### 6-04-release-commit.png — 4.а Реліз комітом

**Що робимо:** Змінюємо тег образу api з 1.1.0 на 1.2.0, комітимо і пушимо.

**Перед скріном:** тег `1.1.0`, Rollout Healthy, `git status` чистий.

**Команди:**
```bash
cd ~/PycharmProjects/DevOps-gitops
sed -i '' '/^  api:$/,/tag:/s/tag: "1.1.0"/tag: "1.2.0"/' charts/booking/values-minikube.yaml
git diff -U0 charts/booking/values-minikube.yaml
git commit -am "Release booking api 1.2.0"
git push
argocd app get booking --refresh | grep -E "Sync Status|Health Status"
```

**Має бути видно:** `-    tag: "1.1.0"`, `+    tag: "1.2.0"` (один рядок, тег worker не змінено); коміт і `main -> main`; `OutOfSync from main (...)`, `Healthy`.

**Не має бути видно:** токена в URL `git push`, паролів admin Argo CD, сповіщень IDE.

**Підпис:** Рис. 4. Реліз 1.2.0: зміна тегу, коміт і push у main

**Що видно:**

### 6-05-paused.png — 4.б Пауза на 20%

**Що робимо:** Чекаємо на першу ручну паузу і показуємо стан релізу в CLI та Argo CD.

**Перед скріном:** блок 6-04 виконано.

**Команди:**
```bash
cd ~/PycharmProjects/DevOps-gitops
argocd app wait booking --suspended --timeout 120 | grep -E "^(Sync|Health) Status"
kubectl argo rollouts get rollout booking-api -n coworking-booking
argocd app get booking | grep -E "Sync Status|Health Status"
```

**Має бути видно:** `Health Status: Suspended`; `Status: ॥ Paused`, `Message: CanaryPauseStep`, `Step: 1/6`, `SetWeight: 20`; образи `1.1.0 (stable)` і `1.2.0 (canary)`; 1 под canary і 4 поди stable.

**Не має бути видно:** паролів admin Argo CD, токенів, сповіщень IDE.

**Підпис:** Рис. 5. Реліз на ручній паузі: 1 под canary і 4 поди stable, застосунок Suspended

**Що видно:**

### 6-06-ratio.png — 4.в Відповіді обох версій

**Що робимо:** Надсилаємо 40 запитів із пода в кластері через Service і рахуємо відповіді кожної версії.

**Перед скріном:** реліз на паузі (6-05).

**Команди:**
```bash
cd ~/PycharmProjects/DevOps-gitops
kubectl -n coworking-booking run curl --rm -i --restart=Never --image=curlimages/curl -- sh -c 'for i in $(seq 1 40); do curl -s http://booking-api/api/health/; echo; sleep 0.3; done' | grep -o '"version":"[^"]*"' | sort | uniq -c
```

**Має бути видно:** два рядки `uniq -c`: більшість `"version":"1.1.0"` і приблизно 8 із 40 `"version":"1.2.0"` (на репетиції було 30 і 10; частка випадкова). Попередження kubectl про запис сесії — нормальне.

**Не має бути видно:** паролів admin Argo CD, токенів, сповіщень IDE.

**Підпис:** Рис. 6. Цикл із 40 запитів усередині кластера: відповідають обидві версії

**Що видно:**

### 6-07-ui-suspended.png — 4.г Інтерфейс Argo CD на паузі

**Що робимо:** Відкриваємо https://localhost:8080 → `booking` і показуємо дерево з Rollout на паузі.

**Перед скріном:** реліз на паузі, **promote ще не виконано**. За потреби натиснути Refresh у UI.

**Команди:** — (знімок вікна браузера: сторінка застосунку `booking`, вид Tree).

**Має бути видно:** `Suspended`, `Synced` до `main` з тим самим комітом, що в 6-05; Rollout `booking-api` у стані Suspended; два ReplicaSet: у canary 1 под, у stable 4; Deployment `booking-worker`, `booking-postgres`; немає Deployment `booking-api`.

**Не має бути видно:** Settings → Accounts, збережених паролів браузера, панелі з токеном, інших вкладок з особистими даними.

**Підпис:** Рис. 7. Дерево застосунку booking в Argo CD: Rollout на ручній паузі

**Що видно:**

### 6-08-promote.png — 4.д Promote і завершення релізу

**Що робимо:** Виконуємо promote на ручній паузі, чекаємо проходження 60%, 80%, 100% і повторюємо цикл запитів.

**Перед скріном:** зроблено 6-07, реліз ще на паузі.

**Команди:**
```bash
cd ~/PycharmProjects/DevOps-gitops
kubectl argo rollouts promote booking-api -n coworking-booking
kubectl argo rollouts status booking-api -n coworking-booking --timeout 300s
kubectl -n coworking-booking run curl --rm -i --restart=Never --image=curlimages/curl -- sh -c 'for i in $(seq 1 40); do curl -s http://booking-api/api/health/; echo; sleep 0.3; done' | grep -o '"version":"[^"]*"' | sort | uniq -c
```

**Має бути видно:** `rollout 'booking-api' promoted`; рядки `Progressing`, `Paused - CanaryPauseStep` і в кінці `Healthy`; `40 "version":"1.2.0"`.

**Не має бути видно:** паролів admin Argo CD, токенів, сповіщень IDE.

**Підпис:** Рис. 8. Promote: реліз дійшов до Healthy, усі відповіді від 1.2.0

**Що видно:**

---

## 5. Скасування релізу

### 6-09-abort-release.png — 5.а Реліз 1.3.0 і abort

**Що робимо:** Запускаємо ще один реліз, чекаємо паузи і скасовуємо його на проміжному кроці.

**Перед скріном:** тег `1.2.0`, Rollout Healthy (6-08).

**Команди:**
```bash
cd ~/PycharmProjects/DevOps-gitops
sed -i '' '/^  api:$/,/tag:/s/tag: "1.2.0"/tag: "1.3.0"/' charts/booking/values-minikube.yaml
git commit -am "Release booking api 1.3.0"
git push
argocd app get booking --refresh | grep -E "Sync Status|Health Status"
argocd app wait booking --suspended --timeout 120 | grep -E "^(Sync|Health) Status"
kubectl argo rollouts abort booking-api -n coworking-booking
kubectl argo rollouts get rollout booking-api -n coworking-booking
```

**Має бути видно:** коміт `Release booking api 1.3.0`; `Health Status: Suspended`; `rollout 'booking-api' aborted`; `Status: ✖ Degraded`, `RolloutAborted`, `Step: 0/6`, `SetWeight: 0`, `1.2.0 (stable)`. Canary-под може ще бути `Terminating` кілька секунд.

**Не має бути видно:** токена в URL `git push`, паролів admin Argo CD, сповіщень IDE.

**Підпис:** Рис. 9. Реліз 1.3.0 скасовано на паузі: Rollout Degraded, stable лишається 1.2.0

**Що видно:**

### 6-10-synced-degraded.png — 5.б Synced + Degraded

**Що робимо:** Показуємо стан застосунку в Argo CD і те, що користувачі отримують стару версію.

**Перед скріном:** блок 6-09 виконано, revert ще не зроблено.

**Команди:**
```bash
cd ~/PycharmProjects/DevOps-gitops
argocd app wait booking --degraded --timeout 60 | grep -E "^(Sync|Health) Status"
argocd app get booking | grep -E "Sync Status|Health Status"
kubectl -n coworking-booking run curl --rm -i --restart=Never --image=curlimages/curl -- sh -c 'for i in $(seq 1 40); do curl -s http://booking-api/api/health/; echo; sleep 0.3; done' | grep -o '"version":"[^"]*"' | sort | uniq -c
```

**Має бути видно:** `Sync Status: Synced to main (...)`, `Health Status: Degraded`; `40 "version":"1.2.0"`.

**Не має бути видно:** паролів admin Argo CD, токенів, сповіщень IDE.

**Підпис:** Рис. 10. Synced + Degraded: у Git тег 1.3.0, працює 1.2.0

**Що видно:**

### 6-11-ui-degraded.png — 5.в Інтерфейс Argo CD у Degraded

**Що робимо:** Відкриваємо сторінку `booking` в Argo CD і показуємо Degraded.

**Перед скріном:** revert ще не зроблено. За потреби натиснути Refresh у UI.

**Команди:** — (знімок вікна браузера: сторінка застосунку `booking`, вид Tree).

**Має бути видно:** `Degraded`, `Synced` до коміту `Release booking api 1.3.0`; Rollout `booking-api` у стані Degraded; 5 подів у ReplicaSet версії 1.2.0.

**Не має бути видно:** Settings → Accounts, збережених паролів браузера, панелі з токеном, інших вкладок з особистими даними.

**Підпис:** Рис. 11. Застосунок booking в інтерфейсі Argo CD: Synced і Degraded

**Що видно:**

### 6-12-revert.png — 5.г Завершення відкату через Git

**Що робимо:** Скасовуємо коміт із 1.3.0 через `git revert` і перевіряємо, що застосунок знову здоровий.

**Перед скріном:** зроблено 6-11, застосунок Synced + Degraded.

**Команди:**
```bash
cd ~/PycharmProjects/DevOps-gitops
git revert $(git log --grep="Release booking api 1.3.0" --format=%h -1) --no-edit
git push
argocd app get booking --refresh | grep -E "Sync Status|Health Status"
argocd app wait booking --sync --health --timeout 180 | grep -E "^(Sync|Health) Status"
kubectl argo rollouts get rollout booking-api -n coworking-booking
argocd app history booking | tail -4
```

**Має бути видно:** коміт `Revert "Release booking api 1.3.0"`; спершу `OutOfSync`, `Degraded`, потім `Synced`, `Healthy`; `Status: ✔ Healthy`, `Step: 6/6`, `1.2.0 (stable)`, 5 подів; в історії останні ревізії — коміт релізу і коміт revert.

**Не має бути видно:** токена в URL `git push`, паролів admin Argo CD, сповіщень IDE.

**Підпис:** Рис. 12. Відкат через git revert: Healthy на версії 1.2.0, історія ревізій

**Що видно:**

---

## Після зйомки (не знімається)

Повернути стабільний стан для захисту: тег `1.1.0`, щоб наживо показати реліз 1.2.0, реліз 1.3.0 і revert.
Повернення теж іде через кроки canary, тому потрібен promote.

```bash
cd ~/PycharmProjects/DevOps-gitops
sed -i '' '/^  api:$/,/tag:/s/tag: "1.2.0"/tag: "1.1.0"/' charts/booking/values-minikube.yaml
git commit -am "Reset booking api to 1.1.0 for live demo"
git push
argocd app get booking --refresh | grep -E "Sync Status|Health Status"
argocd app wait booking --suspended --timeout 120 | grep -E "^(Sync|Health) Status"
kubectl argo rollouts promote booking-api -n coworking-booking
kubectl argo rollouts status booking-api -n coworking-booking --timeout 300s
argocd app list
git status
```

Очікування: Rollout `Healthy`, усі застосунки `Synced  Healthy`, `git status` чистий (скріни в `reports/screens/лаб6/` ще
не закомічені — їх закомічу разом із PDF). Якщо 8081 перестав відповідати, перезапустіть термінал №3, тоді
`curl -s localhost:8081/api/health/` покаже `"version":"1.1.0"`.
