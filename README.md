# Cloud gateway — production reference

Этот каталог закрывает архитектурный пробел «облачный Тренер на любом Android с интернетом» на уровне исходника.

## Что делает шлюз

- хранит ключ/авторизацию Gemini **только на сервере** через secret `GEMINI_API_KEY`;
- отдаёт `GET /healthz` для process/liveness probe без обращения к провайдеру и `GET /v1/status` для реального provider/auth/model readiness probe;
- принимает `POST /v1/coach` только с gateway-схемой `moy-trainer-cloud-gateway-v2`; тренировочный контекст допускается только по **точному** allowlist schema (`moy-trainer-cloud-context-v2`) и разрешённым верхнеуровневым разделам; неизвестная/будущая schema не проходит по одному префиксу;
- сам задаёт системный контракт Тренера и включает Google Search grounding;
- возвращает envelope `moy-trainer-cloud-coach-response-v2`: `{text, sources, modelIdentity, planRevision, requestId}`;
- строит `decisionId` и `revisionId` детерминированно из model/input snapshot/patch; `PlanRevision/v2` всегда содержит `directionId`, `baseRevisionId`, `inputSnapshotIdentity`, `effectiveFrom`, reason и model identity;
- ограничивает размер запроса и ответа, ставит `no-store`, `nosniff`, `no-referrer`, запрещает frame embedding, ведёт только технический журнал без текста вопроса и без тренировочного payload;
- имеет процессный аварийный rate-limit. Он не заменяет распределённый лимит на reverse proxy / API gateway;
- кратко кэширует реальный `/v1/status` provider probe (по умолчанию 10 с для успеха и 3 с для ошибки), чтобы несколько одновременных клиентов не создавали шквал одинаковых запросов к провайдеру;
- поддерживает необязательный server-side origin guard: если задан `GATEWAY_EDGE_SHARED_SECRET`, доверенный edge добавляет `X-Moy-Edge-Key`, а прямой обход edge к `/v1/status` и `/v1/coach` получает 403; Android этот secret не знает;
- не имеет API для записи тренировочного state: optional `PlanRevision/v2` валидирует и атомарно применяет Android-приложение; stale response, смена direction и несовместимое состояние отклоняются локально.

## Как Android получает адрес

Секрет в APK не кладётся. При сборке задаётся только публичный HTTPS URL:

```bash
./gradlew :app:assembleProductionRelease \
  -PMOY_TRAINER_CLOUD_GATEWAY_URL=https://trainer.example.com
```

Либо используется переменная окружения `MOY_TRAINER_CLOUD_GATEWAY_URL`.

Если production APK собран без URL, приложение показывает `CLOUD_SERVICE_UNCONFIGURED` и продолжает работать локально; оно **не просит обычного пользователя получить ключ Google**. Прямой Gemini-ключ разрешён только test/debug сборке как диагностический fallback.


## Проверка без развёртывания

`python test-v17-cloud-gateway-contract.py` проверяет статический security/contract слой.
`python test-v17-cloud-gateway-runtime.py` исполняет настоящие endpoint-функции через минимальный тестовый Flask surface и подменённый provider transport. Это позволяет проверить status/error mapping, schema allowlist, размеры, rate limit, grounding/source filtering и безопасные заголовки даже в build-среде без установленного Flask. Этот тест **не является network/deployment PASS**.

## Разделение liveness и readiness

- `/healthz` отвечает только за то, что процесс шлюза жив; он не требует `GEMINI_API_KEY` и не делает внешний запрос. Его можно использовать для liveness/container health check.
- `/v1/status` проверяет provider credential + модель и только его успешный ответ `status=ready` может приводить Android в `CLOUD_READY`.

## Что обязательно перед production PASS

1. Развернуть шлюз за HTTPS и положить `GEMINI_API_KEY` в server-side secret manager/environment, не в репозиторий.
2. На внешнем reverse proxy / API gateway включить распределённые лимиты запросов и бюджета, защиту от массового злоупотребления и TLS. Встроенный процессный лимит — только второй рубеж, а не production-защита сам по себе. Если origin доступен отдельно, задать `GATEWAY_EDGE_SHARED_SECRET` и заставить edge добавлять `X-Moy-Edge-Key` только на серверной стороне; подробности — `deployment/REVERSE_PROXY_CONTRACT.md`.
3. Не делать Google Play / Gemini Nano / конкретный NPU обязательным условием доступа: клиенту достаточно поддерживаемого Android и HTTPS.
4. Проверить `healthz`, `status`, обычный диалог, grounding, structured PlanRevision, stale/CAS rejection, rollback, отклонение неизвестной schema/разделов контекста, offline и 429/5xx.
5. Проверить, что ни provider key, ни серверные секреты не попадают в APK, backup, логи и transfer archive.

Этот каталог — **reference implementation, не свидетельство развёрнутого production-сервиса**.
