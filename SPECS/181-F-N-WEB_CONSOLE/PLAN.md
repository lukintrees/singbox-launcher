# PLAN 181 — LX Web Console

Как строим. Решения, принятые здесь сверх SPEC, помечены **[решение]**.

---

## 1. Слои и файлы

```
core/headless/            ← новый пакет: «голова без GUI»
  presenter.go            интерфейс Presenter + NullPresenter
  app.go                  App: сервисы (state, build, registry, registry машин,
                          TransportPool, lxdclient) без Fyne и без окон
  state_access.go         локальный и per-machine доступ к state.json (мувтекс,
                          schema-гейт, dirty-маркеры) — вынести из debugapi,
                          чтобы им пользовались и lxweb, и debugapi

core/lxweb/               ← новый пакет: HTTP-фасад
  server.go               сервер, роутинг /api/v1, /events, /ui, auth
  auth.go                 bearer + cookie-сессия, argon2id-пароль, rate-limit
  events.go               SSE-мультиплексор: подписка на gRPC-стримы демона /
                          пул транспортов, drop-oldest, топики
  dto.go                  типизированные DTO (то, что видит браузер)
  api_daemon.go           /api/v1/daemon/*
  api_machines.go         /api/v1/machines/*
  api_state.go            /api/v1/state/*
  api_registry.go         /api/v1/registry/*  (схемы, формы, коды warning'ов)
  api_parse.go            /api/v1/parse, /api/v1/emit   (движок linkmap)
  api_build.go            /api/v1/build/preview, /api/v1/deploy
  api_service.go          /api/v1/service/*   (рецепты, команды, лог, паспорт)
  api_logs.go             /api/v1/logs/*, /api/v1/debug/*
  ui/                     go:embed статика (см. §6)
    index.html app.js fields.js theme.css …

core/debugapi/            ← без изменений по поведению; внутри переводится на
                            core/headless/state_access.go (минус дубль)

ui/                       ← Fyne остаётся; UIService за интерфейсом Presenter
```

**[решение]** Ничего из `core/lxweb` не импортирует `ui/` и `internal/fyne*`.
`core/lxweb` импортирует только `core/**`, `internal/lxdclient`, `contract`.

---

## 2. Волна W0 — `core/headless` + `Presenter`

Цель: бинарь, который **компилируется без Fyne** и умеет всё, кроме окон.

1. `Presenter` — минимальный интерфейс по фактическим вызовам (152 упоминания
   `UIService` в 18 файлах `core`):

```go
type Presenter interface {
    // Обновления «экрана»: десктоп — fyne.Do, веб — publish в SSE, headless — no-op.
    UpdateCoreStatus(func())
    UpdateConfigStatus(func())
    RefreshAPI(func())
    AutoPing(func())
    // Взаимодействие: десктоп — диалог/буфер/трей, веб — toast/SSE-запрос.
    Notify(level Level, title, body string)
    CopyToClipboard(text string) bool
    Confirm(title, body string, onYes func())   // асинхронно
    ShowMainWindow()
    Tray(menu []MenuItem)
}
```

2. `fynePresenter` — сегодняшний `UIService` под этим интерфейсом (обёртка, не
   переписывание): `core/uiservice` остаётся, но `core` видит только интерфейс.
3. `webPresenter` — публикует в SSE-топик `toast`/`confirm`; ответ браузера
   приходит отдельным `POST /api/v1/ui/answer`, поэтому `Confirm` асинхронный
   (это и сегодня так: диалог не блокирует).
4. Механическая замена в 18 файлах: `ac.UIService.X` → `ac.Presenter.X`.
   Файлы: `controller.go`, `elevation.go`, `tray_menu.go`, `ui_watchdog.go`,
   `taildrop*.go`, `services/api_service.go`, `daemon_manager.go`, `rebuild.go`.
5. `main.go`: флаги

```
--web                 поднять lxweb рядом с Fyne (127.0.0.1:9264)
--web-only            без Fyne; требует daemon-движок
--listen  addr        (только с --web*) по умолчанию 127.0.0.1:9264
--tls-cert/--tls-key  обязательны, если --listen не loopback
--web-password-stdin  пароль оператора из stdin (не в argv!)
```

6. **Проверка волны:** `go list -deps ./ | grep fyne` пуст для `-tags headless`;
   `go build ./...` без тегов — зелёный; десктоп руками (владелец).

**[решение]** `--web-only` при отсутствии демона не «падает красиво», а
отказывается стартовать с текстом, содержащим готовую команду установки службы
(SPEC 161-рецепты уже есть: `core.DaemonInstallCommand`).

---

## 3. Волна W1 — демон и машины

### 3.1 `/api/v1/daemon/*` (локальный демон; в `--web-only` — основной экран)

| Метод | Путь | Источник в коде |
|---|---|---|
| GET | `/api/v1/daemon` | паспорт + статус + вердикт службы → `DaemonStatus` (debugapi/daemon_endpoints.go) |
| POST | `/api/v1/daemon/start` \| `/stop` \| `/rollback` | `lxdclient.Start/Stop/Rollback` |
| POST | `/api/v1/daemon/apply` | `lxdclient.Apply` + `ApplyError` → 422 с причиной |
| GET | `/api/v1/daemon/config` | `ActiveConfig` |
| GET | `/api/v1/daemon/commands` | `CommandsCtx` (имя, состояние, exit_code, хвост лога) |
| POST | `/api/v1/daemon/commands/{name}` | `RunCommandCtx`; **никогда** не исполняет sudo |
| GET | `/api/v1/daemon/clients` | `ClientsInfo` |
| PUT/DELETE | `/api/v1/daemon/clients/{key}/label` | `SetClientLabel` / `DeleteClientLabel` |
| GET | `/api/v1/daemon/resources` | `Resources` |
| GET | `/api/v1/daemon/wire` | `lxdclient` wire-журнал (метаданные, без тел) |

### 3.2 `/api/v1/machines/*`

Зеркало `/remote/machines/*` из debugapi, но: без `ui/*`-override (это
десктопная концепция), с DTO и с `paired_at`, `last_error` в одном объекте
строки списка — списку не нужно N запросов.

Плюс два, которых в debugapi нет:

| Метод | Путь | Зачем |
|---|---|---|
| GET | `/api/v1/machines/{id}/row` | одна строка списка: reachable, core_status, version, outdated, last_error, deploy_sha — один запрос на машину вместо пяти |
| GET | `/api/v1/machines/{id}/service` | раннбук SPEC 161: диагноз + шаги-рецепты для этой машины (init, пути из паспорта) |

### 3.3 SSE `/events`

```
GET /events?topics=status,proxies,log,conns,host,dns,toast&machine=<id>
→ text/event-stream
event: status
id: 1284
data: {"running":true,"uptime_s":9321,"outbound":"vless-3"}
```

- Топик → подписка: `status` → `daemon.StartedService/SubscribeStatus`,
  `proxies` → `SubscribeGroups` + `SubscribeOutbounds`, `log` → `SubscribeLog`,
  `conns` → `SubscribeConnections`, `host` → admin REST `/admin/host` (poll 2s),
  `dns` → `SubscribeDNSQueries`, `service` → `SubscribeServiceStatus`.
- Один `TransportPool`-транспорт на машину уже есть; SSE **не** создаёт
  соединений на браузер — только читает общий стрим.
- Backpressure: буфер 256 событий на топик; переполнение → drop-oldest и
  комментарий `: dropped N` (клиент показывает «пропуск»).
- Авторизация: EventSource не умеет заголовки → сессионная cookie (§4).

---

## 4. Аутентификация

```
POST /auth/login   {password}        → Set-Cookie: lx_session=…; HttpOnly; SameSite=Strict
POST /auth/token   {token}           → то же (bearer-токен из settings.json / --web-token)
POST /auth/logout
GET  /auth/me                        → {mode, user, expires_at}
```

- Пароль оператора: `argon2id`, хеш в `settings.json` (`web_password_hash`),
  задаётся `singbox-launcher --web-passwd` (интерактивно) или
  `--web-password-stdin` (docker secret). **[решение]** никакого пароля в argv и
  в env по умолчанию.
- Сессия: HMAC-подписанный токен (ключ — `web_session_key` в settings,
  генерится один раз), TTL 12 ч, привязка к `User-Agent`-хешу.
- `X-LX-Web: 1` обязателен на всех мутациях (жалоба «нет заголовка» = вероятный
  CSRF; браузерная статика ставит его автоматически).
- Rate-limit: 5 попыток/мин на IP для `/auth/*`, 1/сек на `/deploy`.

---

## 5. Фасад состояния (W2) и реестра (W3)

### 5.1 `/api/v1/state/*`

Дерево, а не плоский список. Форма ответа:

```json
{ "rev": 42,
  "tree": [
    {"kind":"folder","id":"01J…","name":"Work","enabled":true,"nodes":4,"replace":"work-fold"},
    {"kind":"subscription","id":"01K…","name":"Liberty","url":"https://…","nodes":292,
     "identity":{"ua":"Happ"},"last_error":null,"updated_at":"2026-11-12T09:10:00Z"},
    {"kind":"server","id":"01M…","tag":"vless-3","scheme":"vless","url":null,
     "subtitle":"vless·ws·TLS+Reality","warnings":["reality_fp_not_chrome"],"disabled":false}
  ],
  "directions":[…], "rules":[…], "dns":{…}, "vars":[…], "target":{…} }
```

- `rev` — монотонный счётчик записи состояния. **Каждая мутация присылает
  `If-Match: rev`**; рассинхрон → `409` с текущим `rev` и диффом (это же
  закроет давнюю гонку из `debugapi/state_endpoints.go` §guardStateSchema —
  там schema-гейт есть, а version-гейта нет).
- Мутации по одному объекту, как в SPEC 160: `POST /state/nodes`,
  `PATCH /state/nodes/{tag}`, `DELETE /state/nodes?tag=&folder=`,
  `POST /state/subscriptions`, `POST /state/subscriptions/{id}/refresh`,
  `POST /state/folders`, `PATCH /state/chains/{tag}`, `PATCH /state/directions/{name}`,
  `PATCH /state/vars`, `PATCH /state/rules`, `PATCH /state/dns`.
- Каждая мутация запускает тот же «save → dirty-маркеры → rebuild» путь, что
  кнопка Конфигуратора (SPEC 045/050), и возвращает
  `{"rev":43,"config_rebuilt":true,"report":[…]}`.

### 5.2 `/api/v1/registry/*` — то, что делает форму данными

| Путь | Что отдаёт |
|---|---|
| `GET /registry/schemes` | 20 схем: `scheme`, `kind` (outbound/endpoint/group), `singbox_type`, `aliases`, `title_en/ru` |
| `GET /registry/schemes/{s}` | `blocks[]`, `choices[]`, `fields[]` в порядке `order`: `{path, block, kind (string/enum/bool/int/duration/list/range), values, secret, required, visible_when, implies, min, max, variant, variant_of, discriminator, label_en, label_ru, hint_en, hint_ru}` |
| `GET /registry/warnings` | 120 кодов: `severity`, `title_en/ru`, `text_en/ru`, `cause_en/ru`, `params` |
| `GET /registry/allowlists` | `fingerprints`, `alpn`, `curve_preferences`, `packet_encoding`, … |
| `GET /registry/vars` | типизированные vars шаблона (enum/dns_server/outbound + whitelists) |

`fields[]` собирается **функцией реестра** (`FormFields` + `FormState` для
`visible`/`required`/`choices` на конкретном теле) — сервер ничего не считает
сам. Подписи: `label_en/ru` из реестра; до волны R4 (см. SPEC §5) сервер
берёт их из генератора `contract/registry/labels.json`, собранного из
`ui/configurator/business/node_form_labels.go` (скрипт `tools/labelsgen`,
разовый).

### 5.3 `/api/v1/parse` и `/api/v1/emit` — «вставил текст → увидел ноды»

```
POST /api/v1/parse   {"input":"<share-link|base64|singbox|xray|conf|vpn://>",
                      "locale":"ru"}        (≤ 2 МиБ)
→ {"nodes":[{kind,scheme,label,entry,warnings:[{code,path,value,severity,
              title,text,cause}]}],
   "dropped":[{ref,index,code,reason}], "meta":{"kind":"uri_list","count":12}}
```

Исполняется `linkmap` + `registry` (`nodeflow.Sanitize`), тексты warning'ов —
из `warnings.json` по локали запроса. Это тот же путь, что у корпуса
`contract/corpus`, поэтому расхождений быть не может: движок один.

```
POST /api/v1/emit    {"scheme":"vless","body":{…}} → {"uri":"vless://…","warnings":[…]}
```

---

## 6. Клиент (W1–W5, растёт по волнам)

Файлы (`core/lxweb/ui/`), без сборки и без node:

```
index.html        оболочка: nav + <main> + тосты/модалки
app.js            роутер (hash → вкладка), стор, fetch-обёртка с If-Match
events.js         EventSource, reconnect, индикатор «пропуск»
fields.js         рендерер поля по DTO реестра (один на все 20 схем)
nodes.js          дерево источников, drag-reorder, карточка ноды
forms.js          форма узла: блоки-аккордеоны, группы выбора, варианты
rules.js dns.js   правила/DNS (preset-строки, drag, текстовый режим)
service.js        раннбук: диагноз, шаги-команды с ⧉ и подсветкой
logs.js           лог ядра/машины, соединения, DNS, телеметрия хоста
build.js          preview конфига + отчёт сборки (по `BuildReportEntry`)
theme.css
```

Правила вёрстки:
- grid 12 колонок, ломается в одну колонку на `< 900px`; на `< 600px` —
  нижняя навигация.
- Тач-цели ≥ 44px, все действия доступны без hover.
- Тёмная/светлая — `prefers-color-scheme` + переключатель в localStorage.
- Ни одной иконки-картинки: SVG inline (как в `ui/icons`), чтобы не тащить
  ассеты.

---

## 7. Прототип

`prototype/index.html` — **кликабельный макет без бэкенда** (открывается
двойным щелчком, работает и как `file://`): демонстрация раскладки, вкладок,
формы узла из реестра и живых данных на синтетике. В прототип вшиты реальные
куски `contract/registry` (vless, wireguard, tls, transports, warnings) —
поэтому форма в прототипе **не нарисована руками**, а собрана тем же правилом,
что будет на сервере.

Прототип не поставляется в релиз; он в SPECS, как и положено макету.

---

## 8. Проверки

- Локально: `go build ./...`, `go build -tags headless ./core/lxweb/...`,
  `go run ./tools/win7guard`, один именованный тест на волну
  (`TestLXWebStateRevConflict`, `TestLXWebEventMultiplexerDropOldest`, …).
- Полный прогон — CI (`gh workflow run ci.yml --ref develop -f run_mode=tests`).
- Контракт: `contract` не меняется в W0–W3; R4 (подписи в реестр) — отдельный
  бамп `contract/VERSION` + `TASKS_LXBOX.md`.
- UI: тестами не покрывается (CONSTITUTION §8.1) — показывает владелец.

---

## 9. Порядок работ и размер

| Волна | Кто | Оценка |
|---|---|---|
| W0 `Presenter` + `--web*` + статика-заглушка | 1 | ~1,5 нед. |
| W1 демон + машины + SSE | 1 | ~2 нед. |
| W2 state CRUD + rev-гейт | 1 | ~2 нед. |
| W3 реестр-формы + parse/emit | 1 | ~2 нед. |
| W4 build/deploy/backup | 1 | ~1 нед. |
| W5 безопасность/LAN/TLS/l10n/мобильное | 1 | ~1 нед. |

Итого ~9–10 недель одного исполнителя; каждая волна — самостоятельный
показываемый результат. W0 и W1 полезны уже сами по себе: «консоль демона с
телефона» закрывает SPEC 161-боль без полного конфигуратора.
