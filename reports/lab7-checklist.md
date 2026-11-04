# Лабораторна №7 — чекліст скриншотів

15 скриншотів, пункти 1–6 методички. Кожен PNG лежить у `reports/screens/лаб7/`.
`reports/build_report.py` бере звідси «Що робимо», «Підпис» і «Що видно» для PDF.
«Що видно» заповнюється після перегляду справжніх скриншотів.

## 1. Стек моніторингу через Argo CD

### 7-01-monitoring-apps.png — 1. Застосунки стеку й поди monitoring

**Що робимо:** Перевіряємо, що kube-prometheus-stack, Loki і Alloy встановлені з Git окремими Application, і всі їхні поди працюють.

**Підпис:** argocd app list і поди простору monitoring.

**Команди:**
```bash
argocd app list | awk '{print $1, $3, $5, $6}' | column -t
kubectl -n monitoring get pods
```

**Що видно:** Усі шість Application, серед них `kps`, `loki` і `alloy`, мають стан `Synced` і `Healthy`. Поди Prometheus, Grafana, Alertmanager, Loki й Alloy у просторі `monitoring` працюють, перезапусків немає.

## 2. Метрики застосунку

### 7-02-targets-up.png — 2. Цілі booking-api у Prometheus

**Що робимо:** Показуємо, що ServiceMonitor із чарту привів поди api у цілі Prometheus.

**Підпис:** Prometheus Targets, цілі booking-api у стані UP.

**Що видно:** ServiceMonitor `coworking-booking/booking-api` дає 5 цілей із 5 у стані `UP`, по одній на кожен под api. Prometheus знімає метрики з порту 8000 за шляхом `/metrics`, панель браузера обрізано.

### 7-03-promql.png — 3. Запит RPS у Prometheus

**Що робимо:** Виконуємо PromQL до власної метрики `http_requests_total` під навантаженням генератора `load`.

**Підпис:** Графік RPS застосунку в Prometheus.

**Що видно:** Коли запущено генератор `load`, крива росте від 0 і тримається близько 45 запитів на секунду. Просідання майже до 38 припадає на перший поганий реліз, після нього значення повертається.

## 3. Логи

### 7-04-json-logs.png — 4. JSON-логи запитів

**Що робимо:** Показуємо структурований лог запитів, який Alloy збирає в Loki.

**Підпис:** kubectl logs поду api, рядки JSON.

**Команди:**
```bash
kubectl -n coworking-booking logs "$(kubectl -n coworking-booking get pods -l app.kubernetes.io/component=api -o name | head -1)" --tail=6
```

**Що видно:** Кожен запит пише один рядок JSON із полями `method`, `route`, `status`, `duration_ms` і `version`. Завдяки цьому Loki може фільтрувати за цими полями без окремих правил розбору.

### 7-05-loki-5xx.png — 5. LogQL по 5xx у Grafana Explore

**Що робимо:** Фільтруємо логи всіх подів api за статусом, розбираючи JSON прямо в запиті.

**Підпис:** Grafana Explore, запит за status >= 500.

**Що видно:** Запит `| json | status >= 500` знаходить 402 рядки за 30 хвилин, усі рівня error і згруповані в часі першого поганого релізу. Поля рядка видно окремо, зокрема `status: 500` і `version: 1.6.0`.

## 4. Дашборд

### 7-07-dashboard-git.png — 6. Дашборд лежить у Git

**Що робимо:** Показуємо, що дашборд приходить із Git як ConfigMap із міткою, а не створений вручну в Grafana.

**Підпис:** ConfigMap дашборду і історія файлу в Git.

**Команди:**
```bash
kubectl get cm -l grafana_dashboard=1 -A | grep -E "NAMESPACE|booking"
git log --oneline -- charts/booking/dashboards/booking.json
```

**Що видно:** ConfigMap `booking-dashboard` із міткою `grafana_dashboard=1` належить простору `coworking-booking`. Файл `charts/booking/dashboards/booking.json` має власну історію в Git, тож дашборд відтворюється з репозиторію.

### 7-06b-dashboard-logs.png — 7. Панель логів Loki на дашборді

**Що робимо:** Відкриваємо восьму панель дашборду, «Логи помилок (Loki)», на час поганого релізу. На загальному кадрі вона нижче екрана.

**Підпис:** Панель «Логи помилок (Loki)» дашборду Booking API.

**Що видно:** Панель показує рядки рівня error: запит зі `status: 500` і `version: 1.6.0`, а поруч рядок Django `Internal Server Error: /api/resources/`.

## 5. Навантаження

### 7-06-dashboard-load.png — 8. Навантаження і збільшення реплік на дашборді

**Що робимо:** Показуємо дашборд за час запуску генератора `load` і зміни реплік із 5 до 8 через Git.

**Підпис:** Дашборд Booking API, 21:14–21:23: старт навантаження і зміна реплік.

**Що видно:** Після запуску `load` RPS росте приблизно від 4 до 46 запитів на секунду, а CPU одного пода зростає майже від 0 до близько 0.05 ядра. Затримка p95 тримається близько 10 мс і коротко піднімається до 18 мс близько 21:20, у час зміни реплік; причину цього підйому окремо не досліджували. Prometheus показав 8 цілей `up` о 21:19:57, а після revert знову 5. Помилок немає, панель 5xx порожня.

### 7-06c-replicas-git.png — 9. Зміна реплік комітом

**Що робимо:** Показуємо, що кількість реплік змінено комітом у Git, а не вручну.

**Підпис:** Коміт зміни реплік і його revert.

**Команди:**
```bash
git log --oneline --grep='8 replicas'
git show 9d996ca -- charts/booking/values-minikube.yaml | grep -E '^[-+] '
```

**Що видно:** Коміт `9d996ca` змінює `api: 5` на `api: 8`, а `f66d90f` його скасовує. Обидві зміни застосував Argo CD, ручного `kubectl scale` не було.

## 6. Автоматичний відкат за метриками

### 7-08-good-release.png — 10. Хороший реліз пройшов аналіз

**Що робимо:** Показуємо реліз 1.5.0: AnalysisRun успішний, Rollout сам дійшов до кінця без ручного promote.

**Підпис:** Rollout Healthy і успішний AnalysisRun.

**Команди:**
```bash
kubectl argo rollouts get rollout booking-api -n coworking-booking | sed -n 1,14p
kubectl -n coworking-booking get analysisrun booking-api-77dd478494-3-2
kubectl -n coworking-booking get analysisrun booking-api-77dd478494-3-2 -o jsonpath='{.status.metricResults[0].phase}{"  count="}{.status.metricResults[0].count}{"  values="}{.status.metricResults[0].measurements[*].value}{"\n"}'
```

**Що видно:** Rollout `Healthy`, крок 5/5, образ 1.5.0 позначено як stable, 5 подів із 5 готові. AnalysisRun `booking-api-77dd478494-3-2` має стан `Successful`: 4 виміри зі значенням 1, ручного promote не було.

### 7-09-bad-analysis.png — 11. Поганий реліз: аналіз провалено

**Що робимо:** Комітимо реліз 1.6.0 з `FAIL_RATE: "0.5"` і чекаємо: AnalysisRun сам падає, Rollout сам робить abort.

**Підпис:** Rollout Degraded і AnalysisRun Failed.

**Команди:**
```bash
kubectl argo rollouts get rollout booking-api -n coworking-booking | sed -n 1,14p
AR=$(kubectl -n coworking-booking get analysisrun --sort-by=.metadata.creationTimestamp -o name | tail -1 | cut -d/ -f2)
kubectl -n coworking-booking get analysisrun $AR
kubectl -n coworking-booking get analysisrun $AR -o jsonpath='{.status.message}{"\n"}{.status.metricResults[0].measurements[*].value}{"\n"}'
```

**Що видно:** Аналіз отримав значення 0.9469 і 0.9011, обидва нижчі за поріг 0.95, тому AnalysisRun `Failed` із причиною `failed (2) > failureLimit (1)`. Rollout сам перейшов у `Degraded` з повідомленням `RolloutAborted`, вага 0, stable лишається 1.5.0, усього 76 с від push.

### 7-10-synced-degraded.png — 12. Synced і Degraded, користувачі на старій версії

**Що робимо:** Показуємо, що Argo CD застосував Git (Synced), здоров'я Degraded, а запити обслуговує stable.

**Підпис:** argocd app get booking і цикл curl.

**Команди:**
```bash
argocd app get booking | grep -E "Sync Status|Health Status"
for i in $(seq 20); do curl -s -m3 localhost:8081/api/health/ | python3 -c 'import json,sys;print(json.load(sys.stdin)["version"])'; done | sort | uniq -c
for i in $(seq 20); do curl -s -m3 -o /dev/null -w "%{http_code}\n" localhost:8081/api/health/; done | sort | uniq -c
```

**Що видно:** Argo CD показує `Synced to main (ba7626c)` і `Degraded`: Git застосовано, але реліз зупинено. Усі 20 запитів відповіли версією 1.5.0, ще 20 запитів повернули код 200, тож користувачі вже на старій версії.

### 7-11-dashboard-errors.png — 13. Помилки на дашборді

**Що робимо:** Дивимось на дашборді 5xx по подах під час поганого релізу.

**Підпис:** Сплеск 5xx лише на canary-поді.

**Що видно:** На панелі «5xx по подах» є лише один под `booking-api-84f5b867f4-fdwnn` із приблизно 3.1–3.8 помилки на секунду. Частка 5xx сягнула близько 9% і впала до 0 після abort. Точок лише дві, бо canary прожив недовго.

### 7-12-loki-new-version.png — 14. Помилки в Loki лише від нової версії

**Що робимо:** Той самий запит по 5xx у Loki, тепер видно поле `version`.

**Підпис:** Grafana Explore, рядки 5xx із version 1.6.0.

**Що видно:** За 5 хвилин знайдено 276 рядків зі статусом 500. Окремий запит до Loki підтвердив, що всі вони від одного поду `booking-api-84f5b867f4-fdwnn` і версії 1.6.0.

### 7-13-revert.png — 15. git revert і повернення до Healthy

**Що робимо:** Abort зупинив трафік, але Git досі каже 1.6.0. Робимо `git revert`, чекаємо Synced і Healthy на 1.5.0.

**Підпис:** Rollout Healthy на 1.5.0 і історія Argo CD.

**Команди:**
```bash
git log --oneline -3
argocd app get booking | grep -E "Sync Status|Health Status"
kubectl argo rollouts get rollout booking-api -n coworking-booking | sed -n 1,10p
argocd app history booking | tail -4 | cut -c1-110
```

**Що видно:** Коміт `6bac534` скасовує `ba7626c`, а наступний коміт змінює лише шпаргалку. Застосунок `Synced` і `Healthy`, Rollout `Healthy` на образі 1.5.0, історія Argo CD містить і поганий реліз (`ba7626c`), і синхронізацію після відкату (`834603c`).
