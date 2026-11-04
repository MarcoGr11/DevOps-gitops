# Observability: шпаргалка для захисту

Стартовий стан: у Git `image.api.tag: "1.5.0"`, `FAIL_RATE: "0"`, аналіз увімкнено, Rollout Healthy, 5 подів,
`root`, `argo-rollouts`, `booking`, `kps`, `loki`, `alloy` усі Synced / Healthy, генератор `load` працює.
Запасні образи: 1.6.0 (поганий реліз), 1.7.0. Очікувані числа взяті з репетиції 2026-11-04; хеші у вас будуть інші.
Усі команди виконуються з `~/PycharmProjects/DevOps-gitops`.

Порядок: Prometheus Targets і метрика → навантаження і дашборд → Loki по 5xx → поганий реліз і автовідкат → git revert.

## 0. Підготовка

Окремі вікна (лишити працювати):

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```
```bash
kubectl -n coworking-booking port-forward svc/booking-api 8081:80
```
```bash
kubectl -n monitoring port-forward svc/kps-grafana 3000:80
```
```bash
kubectl -n monitoring port-forward svc/kps-prometheus 9090:9090
```
```bash
kubectl argo rollouts dashboard
```

| Що | Адреса | Вхід |
|---|---|---|
| Argo CD | https://localhost:8080 | admin, початковий пароль (зміну пароля пропущено) |
| Застосунок | http://localhost:8081/api/health/ | немає |
| Grafana | http://localhost:3000 | admin, пароль із Secret `grafana-admin` |
| Prometheus | http://localhost:9090 | немає |
| Rollouts | http://localhost:3100 | немає |

Основне вікно:

```bash
cd ~/PycharmProjects/DevOps-gitops && git pull
argocd login localhost:8080 --username admin --insecure
argocd app list
kubectl argo rollouts get rollout booking-api -n coworking-booking
kubectl -n default get deploy load
```

Argo CD тут v3.6.1 (клієнт CLI v3.5.6): для цих команд різниця не заважає.

Якщо `load` немає:

```bash
kubectl -n default create deploy load --replicas=3 --image=curlimages/curl -- sh -c 'while true; do curl -s -o /dev/null http://booking-api.coworking-booking.svc/api/resources/; sleep 0.05; done'
```

## 1. Prometheus Targets і запит до метрики

http://localhost:9090/targets → фільтр `booking`.

Очікування: 5 цілей `booking-api`, усі `UP` (ServiceMonitor із чарту, порт `http`, шлях `/metrics`, кожні 15 с).

http://localhost:9090/query:

```promql
sum(rate(http_requests_total{namespace="coworking-booking"}[1m]))
```

Очікування: приблизно 45 запитів на секунду від трьох реплік `load`. Без навантаження: 0.

## 2. Навантаження і дашборд

Grafana → Dashboards → **Booking API**. Дашборд лежить у Git (`charts/booking/dashboards/booking.json`),
у кластері це ConfigMap із міткою `grafana_dashboard: "1"`, Grafana підбирає його сама:

```bash
kubectl get cm -l grafana_dashboard=1 -A | grep booking
```

Очікування під навантаженням: RPS близько 45, p95 близько 0.01 с, CPU подів близько 0.23 ядра (без навантаження: 0.035).

Показати, що зміна реплік іде через Git, а дашборд її бачить (необов'язково):

```bash
sed -i '' 's/^  api: 5$/  api: 8/' charts/booking/values-minikube.yaml
git diff --stat
```

Рядок реплік перевірте очима перед комітом. Коміт, push, refresh: нових подів 8, релізу немає, бо шаблон поду не змінився. Потім `git revert HEAD --no-edit` і push.

## 3. Логи в Loki

Формат рядка:

```bash
kubectl -n coworking-booking logs "$(kubectl -n coworking-booking get pods -l app.kubernetes.io/component=api -o name | head -1)" --tail=2
```

Очікування: JSON із `method`, `route`, `status`, `duration_ms`, `version`. Рядки migrate і gunicorn на старті поду не JSON,
тому `| json` дає на них parse error; фільтр за `status` їх відсікає.

Grafana → Explore → Loki:

```logql
{namespace="coworking-booking",app="api"} | json | status >= 500
```

Очікування у стабільному стані: порожньо за останні хвилини. Загорнути це в метрику: запит 8 нижче.

## 4. Поганий реліз: автоматичний відкат за метриками

Вікно спостереження:

```bash
kubectl argo rollouts get rollout booking-api -n coworking-booking --watch
```

Коміт:

```bash
sed -i '' '/^  api:$/,/tag:/s/tag: "1.5.0"/tag: "1.6.0"/' charts/booking/values-minikube.yaml
sed -i '' '/name: FAIL_RATE/{n;s/value: "0"/value: "0.5"/;}' charts/booking/values-minikube.yaml
git diff
git commit -am "Release 1.6.0 with FAIL_RATE" && git push
argocd app get booking --refresh | grep -E "Sync Status|Health Status"
```

Перебіг (приблизно 2 хвилини від коміту):

1. Крок 20%: один canary-под, пауза 30 с.
2. Запускається AnalysisRun: кожні 30 с частка не-5xx, умова `>= 0.95`, `failureLimit: 1`.
3. У репетиції значення `[1]`, `[0.9005]`, `[0.9050]`: дві невдачі більше за ліміт, AnalysisRun `Failed`.
4. Rollout сам робить abort, canary прибрано, 100% трафіку на 1.5.0. Людина нічого не натискала.

Перший вимір дорівнює 1, бо вікно `[1m]` ще майже не містить canary.

```bash
kubectl argo rollouts get rollout booking-api -n coworking-booking
kubectl -n coworking-booking get analysisrun
kubectl -n coworking-booking describe analysisrun "$(kubectl -n coworking-booking get analysisrun --sort-by=.metadata.creationTimestamp -o name | tail -1 | cut -d/ -f2)" | grep -E "Phase|Message|Value|Failed:|Successful:"
argocd app get booking | grep -E "Sync Status|Health Status"
```

Очікування: `Status: ✖ Degraded`, `Message: RolloutAborted: ...`, `Step: 0/5`, stable 1.5.0.
`Sync Status: Synced`, `Health Status: Degraded`. Це не суперечність: **Git каже 1.6.0, кластер це застосував, але Rollout відмовився його просувати**.
Користувачі при цьому вже на 1.5.0:

```bash
for i in $(seq 20); do curl -s -o /dev/null -w "%{http_code} " localhost:8081/api/health/; done; echo
```

Якщо `curl` не відповідає, port-forward упав разом із подом: перезапустіть його (див. п. 0).

Дашборд: панель 5xx по подах має сплеск **лише на canary-поді**; у репетиції 4.16/с на одному поді.
Loki: запит із п. 3, поле `version` лише `1.6.0`; у репетиції 402 рядки від одного поду.

## 5. Відкат через Git

Abort зупинив трафік, але в Git досі 1.6.0. Git треба привести до правди:

```bash
git revert HEAD --no-edit && git push
argocd app get booking --refresh | grep -E "Sync Status|Health Status"
argocd app wait booking --sync --health --timeout 180
kubectl argo rollouts get rollout booking-api -n coworking-booking
argocd app history booking | tail -5
```

Очікування: `Synced`, `Healthy`, `Status: ✔ Healthy`, образ 1.5.0 (stable), 5 подів, canary-подів немає.

## 6. Алерт MyappHighErrorRate (за запитом викладача)

Правило лежить у `charts/booking/templates/prometheusrule.yaml`, Prometheus його завантажив, стан `inactive`.
Показ алерта в стані `firing` **не репетирувався**. Щоб показати: прибрати крок `analysis` зі стратегії у
`values-minikube.yaml`, зробити реліз 1.7.0 з `FAIL_RATE: "0.5"` комітом, дочекатись canary, відкрити
http://localhost:9090/alerts (спершу Pending, після `for: 1m` Firing). Частка помилок у вікні близько 10% при вазі 20%,
тому поріг 5% спрацьовує. Потім `git revert` обох комітів.

## 10 готових запитів

PromQL (http://localhost:9090/query або Grafana Explore → Prometheus):

1. RPS: `sum(rate(http_requests_total{namespace="coworking-booking"}[1m]))`
2. RPS за маршрутами: `sum by (route) (rate(http_requests_total{namespace="coworking-booking"}[1m]))`
3. Частка 5xx: `sum(rate(http_requests_total{namespace="coworking-booking",status=~"5.."}[1m])) / sum(rate(http_requests_total{namespace="coworking-booking"}[1m]))` (порожньо, поки 5xx не було зовсім: такого ряду ще немає)
4. p95 затримки: `histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{namespace="coworking-booking"}[1m])))`
5. 5xx по подах: `sum by (pod) (rate(http_requests_total{namespace="coworking-booking",status=~"5.."}[1m]))`
6. CPU подів API: `sum(rate(container_cpu_usage_seconds_total{namespace="coworking-booking",pod=~"booking-api.*",container!=""}[1m]))`
7. Пам'ять подів API: `sum(container_memory_working_set_bytes{namespace="coworking-booking",pod=~"booking-api.*",container!=""})`

LogQL (Grafana Explore → Loki):

8. Усі 5xx: `{namespace="coworking-booking",app="api"} | json | status >= 500`
9. Один маршрут: `{namespace="coworking-booking",app="api"} | json | route =~ "api/resources.*"`
10. 5xx за 5 хвилин по подах: `sum by (pod) (count_over_time({namespace="coworking-booking",app="api"} | json | status >= 500 [5m]))`

Додатково, повільні запити: `{namespace="coworking-booking",app="api"} | json | duration_ms > 50`

## Якщо щось пішло не так

- **NaN в аналізі, AnalysisRun не `Failed` і не `Successful`.** Немає трафіку, ділення 0/0. Перевірте `kubectl -n default get deploy load`; без `load` запит порожній.
- **Ціль не UP або цілей немає.** `kubectl -n coworking-booking get servicemonitor`; перевірте, що порт сервісу зветься `http`, а `curl localhost:8081/metrics` віддає метрики.
- **Порожній дашборд.** Діапазон часу (now-15m), datasource, навантаження. `load` не працює: RPS 0, p95 NaN, це нормально.
- **Дашборду немає в Grafana.** `kubectl get cm -l grafana_dashboard=1 -A`; якщо ConfigMap є, подивіться лог `grafana-sc-dashboard` у поді Grafana.
- **Поди Pending.** Пам'ять: `kubectl describe pod <под> | tail` і `kubectl describe node minikube | tail -20`. Кластер на `--memory 8192`; можна тимчасово прибрати `load`.
- **`booking` Progressing на новому кластері.** Немає ingress-контролера: `minikube addons enable ingress`.
- **`Healthy to Degraded` одразу після коміту зі зміною аналізу.** Раніше AnalysisTemplate приходив пізніше за Rollout; тепер у нього sync-wave `-1`.
- **Запит не працює, `connection refused`.** Port-forward упав разом із подом: запустіть його знову. Після цього, якщо `argocd` просить вхід: `argocd login localhost:8080 --username admin --insecure`.

## Чому canary

Canary показує новий образ малій частці користувачів і вимірює його на справжньому трафіку. Blue-green дає миттєве перемикання,
але без поступового вимірювання частка помилок видна лише після переходу всіх. У лабі 6 рішення приймала людина (`promote` / `abort`),
тут рішення приймає AnalysisRun за метрикою.
