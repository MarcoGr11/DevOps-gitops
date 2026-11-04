# Argo Rollouts: шпаргалка для захисту

Стартовий стан: у Git `image.api.tag: "1.1.0"`, Rollout Healthy, 5 подів, `booking`, `root` і `argo-rollouts`
Synced / Healthy. Очікувані виводи нижче взяті з репетиції 2026-11-04; хеші у вас будуть інші.
Усі команди виконуються з `~/PycharmProjects/DevOps-gitops`.

Сценарій: реліз 1.2.0 → пауза → співвідношення версій → promote → реліз 1.3.0 → abort → Synced + Degraded → git revert.

## 0. Підготовка

Окремі вікна (лишити працювати):

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```
```bash
kubectl -n coworking-booking port-forward svc/booking-api 8081:80
```
```bash
kubectl argo rollouts get rollout booking-api -n coworking-booking --watch
```
```bash
kubectl argo rollouts dashboard
```

UI Argo CD: https://localhost:8080. Дашборд Rollouts: http://localhost:3100.

Основне вікно:

```bash
cd ~/PycharmProjects/DevOps-gitops && git pull
argocd login localhost:8080 --username admin --insecure
argocd app list
kubectl argo rollouts get rollout booking-api -n coworking-booking
```

Очікування: усі три застосунки `Synced  Healthy`; `Status: ✔ Healthy`, `Images: ...devops-api:1.1.0 (stable)`, 5 подів.

## 1. Реліз комітом: 1.1.0 → 1.2.0

```bash
sed -i '' '/^  api:$/,/tag:/s/tag: "1.1.0"/tag: "1.2.0"/' charts/booking/values-minikube.yaml
git diff -U0 charts/booking/values-minikube.yaml
git commit -am "Release booking api 1.2.0" && git push
argocd app get booking --refresh | grep -E "Sync Status|Health Status"
```

`sed` змінює лише тег api: у worker такий самий тег `1.1.0`, і його чіпати не можна.

Очікування: `-tag: "1.1.0"`, `+tag: "1.2.0"`; `OutOfSync from main (...)`, `Healthy`.

## 2. Пауза: Suspended

```bash
argocd app wait booking --suspended --timeout 120 | grep -E "^(Sync|Health) Status"
kubectl argo rollouts get rollout booking-api -n coworking-booking
```

Очікування:

```
Sync Status:        Synced to main (...)
Health Status:      Suspended

Status:          ॥ Paused
Message:         CanaryPauseStep
  Step:          1/6
  SetWeight:     20
  ActualWeight:  20
Images:          ghcr.io/marcogr11/devops-api:1.1.0 (stable)
                 ghcr.io/marcogr11/devops-api:1.2.0 (canary)
```

`Step: 1/6` означає, що виконано один крок (`setWeight: 20`) і реліз стоїть на другому, `pause: {}`.
У дереві 1 под canary (revision з 1.2.0) і 4 поди stable. Показати те саме в UI Argo CD (booking, Rollout, Suspended).

## 3. Співвідношення версій усередині кластера

Port-forward прив'язаний до одного пода, тому запити йдуть з пода в кластері через Service:

```bash
kubectl -n coworking-booking run curl --rm -i --restart=Never --image=curlimages/curl -- sh -c 'for i in $(seq 1 40); do curl -s http://booking-api/api/health/; echo; sleep 0.3; done' | grep -o '"version":"[^"]*"' | sort | uniq -c
```

Очікування (на репетиції):

```
  30 "version":"1.1.0"
  10 "version":"1.2.0"
```

Service балансує між подами випадково, тож частка наближена до 20% (у середньому 8 із 40), а не точна.
Попередження `All commands and output from this session will be recorded...` виводить kubectl, воно не заважає.

## 4. Promote

`kubectl argo rollouts status` на ручній паузі чекає до таймауту, тому запускати його треба після promote.

```bash
kubectl argo rollouts promote booking-api -n coworking-booking
kubectl argo rollouts status booking-api -n coworking-booking --timeout 300s
```

Очікування (близько 80 секунд: кроки 60% і 80% із паузами по 30 с, потім 100%):

```
rollout 'booking-api' promoted
Progressing - more replicas need to be updated
Paused - CanaryPauseStep
Progressing - more replicas need to be updated
Paused - CanaryPauseStep
Progressing - more replicas need to be updated
Progressing - updated replicas are still becoming available
Healthy
```

Повторити цикл із розділу 3. Очікування: `40 "version":"1.2.0"`. Застосунок: `Synced`, `Healthy`.

## 5. Новий реліз і abort: 1.2.0 → 1.3.0

```bash
sed -i '' '/^  api:$/,/tag:/s/tag: "1.2.0"/tag: "1.3.0"/' charts/booking/values-minikube.yaml
git commit -am "Release booking api 1.3.0" && git push
argocd app get booking --refresh | grep -E "Sync Status|Health Status"
argocd app wait booking --suspended --timeout 120 | grep -E "^(Sync|Health) Status"
kubectl argo rollouts abort booking-api -n coworking-booking
argocd app wait booking --degraded --timeout 60 | grep -E "^(Sync|Health) Status"
kubectl argo rollouts get rollout booking-api -n coworking-booking
```

Очікування:

```
Sync Status:        Synced to main (...)
Health Status:      Degraded

Status:          ✖ Degraded
Message:         RolloutAborted: Rollout aborted update to revision ...
  Step:          0/6
  SetWeight:     0
  ActualWeight:  0
Images:          ghcr.io/marcogr11/devops-api:1.2.0 (stable)
```

Canary-под 1.3.0 зникає (кілька секунд може бути `Terminating`). Повторити цикл із розділу 3: `40 "version":"1.2.0"`.

**Чому Synced + Degraded.** Synced стосується маніфестів: у Git тег 1.3.0, і в Rollout у кластері теж 1.3.0, тому
розбіжності немає. Degraded стосується роботи: контролер скасував реліз, усі 5 подів працюють на старій 1.2.0, а бажаний
стан із Git не досягнуто. Користувачі в цей час отримують робочу версію. Argo CD не може сам вирішити, що Git неправий,
тому стан лишається таким, доки хтось не виправить Git.

## 6. Завершення відкату через Git

`kubectl argo rollouts retry` не використовуємо: він запустив би той самий скасований 1.3.0 знову.

```bash
git revert $(git log --grep="Release booking api 1.3.0" --format=%h -1) --no-edit
git push
argocd app get booking --refresh | grep -E "Sync Status|Health Status"
argocd app wait booking --sync --health --timeout 180 | grep -E "^(Sync|Health) Status"
kubectl argo rollouts get rollout booking-api -n coworking-booking
argocd app history booking | tail -4
```

Очікування: спершу `OutOfSync`, `Degraded`, далі `Synced`, `Healthy`. Rollout `✔ Healthy`, `Step: 6/6`, `1.2.0 (stable)`.
Кроків canary тут немає: тег у Git знову збігається зі stable-версією, тож контролер одразу вважає реліз завершеним.

## Чому canary, а не blue-green

- API не зберігає стан між запитами, тож обидві версії можуть обслуговувати запити одночасно. Canary показує нову версію
  на реальних запитах малій частці користувачів ще до повного переходу.
- Blue-green на час релізу потребує вдвічі більше подів (10 замість 5), а тут локальний minikube. Canary тримає 5 подів.
- Не потрібні додатковий preview-сервіс і маршрутизатор трафіку: вага задається часткою подів за звичайним Service.
- Обмеження: обидві версії працюють з однією PostgreSQL, тож зміни схеми БД мають бути сумісними зі старою версією.
  Без маршрутизатора вага прив'язана до кількості подів: при 5 репліках крок дорівнює 20%.

## Як змінити довільне значення

Лише через Git: Argo CD із selfHeal поверне ручні зміни назад.

```bash
vim charts/booking/values-minikube.yaml
git commit -am "Опис зміни" && git push
argocd app get booking --refresh | grep -E "Sync Status|Health Status"
```

Що станеться (випливає з шаблонів чарту, на репетиції не перевірялось, крім тегу):

| Значення | Результат |
|---|---|
| `image.api.tag` | новий реліз canary з паузою |
| `config.*` | змінюється checksum у шаблоні пода, тому теж реліз canary; для worker звичайне оновлення Deployment |
| `replicaCount.api` | лише масштабування, без релізу |
| `rollout.strategy.canary.steps` | нові кроки діють з наступного релізу, поди не перезапускаються |
| `image.worker.tag`, `replicaCount.worker` | звичайне оновлення Deployment worker, api не змінюється |
| `version` у `Chart.yaml` | змінюється мітка `helm.sh/chart` у шаблонах подів: новий canary для api і оновлення worker та postgres |

Якщо реліз завис на паузі після такої зміни: `kubectl argo rollouts promote booking-api -n coworking-booking`.

## Якщо щось пішло не так

| Симптом | Що робити |
|---|---|
| `connection refused` на 8080 або 8081 | перезапустити port-forward (розділ 0); 8081 падає, коли зникає под, до якого він прив'язаний |
| `invalid session` / `token is expired` | `argocd login localhost:8080 --username admin --insecure` |
| Застосунок не бачить коміт | `argocd app get booking --refresh` |
| `pods "curl" already exists` | `kubectl -n coworking-booking delete pod curl`, потім повторити цикл |
| Цикл curl повертає порожній результат | прибрати `\| grep ... \| uniq -c`, подивитись сирий вивід |
| Дашборд не відкривається | `pkill -f "rollouts dashboard"`, знову `kubectl argo rollouts dashboard`, http://localhost:3100 |
| `status` завершився з `degraded state` після abort | так і має бути, це не помилка команди |
| minikube зупинено | `minikube start --driver=docker`, далі розділ 0; після старту `booking` може недовго бути `Progressing` |
| Sync: `no matches for kind "Rollout"` | дочекатись Healthy у `argo-rollouts`, потім `argocd app sync booking` (retry теж допомагає сам) |

## Після показу: повернути старт

```bash
sed -i '' '/^  api:$/,/tag:/s/tag: "1.2.0"/tag: "1.1.0"/' charts/booking/values-minikube.yaml
git commit -am "Reset booking api to 1.1.0 for live demo" && git push
argocd app get booking --refresh | grep -E "Sync Status|Health Status"
argocd app wait booking --suspended --timeout 120 | grep -E "^(Sync|Health) Status"
kubectl argo rollouts promote booking-api -n coworking-booking
kubectl argo rollouts status booking-api -n coworking-booking --timeout 300s
```

Повернення на стару версію теж іде через кроки canary, тому потрібен promote.
