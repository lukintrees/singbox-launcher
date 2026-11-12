# TASKS 181 — LX Web Console

Чеклист по волнам. Ничего не начато: статус SPEC — **N**.

## W0 — голова без GUI

- [ ] T0.1 `core/headless.Presenter` + `NullPresenter` (интерфейс по фактическим
      вызовам `UIService`, 152 упоминания в 18 файлах `core`)
- [ ] T0.2 `fynePresenter` — обёртка над `core/uiservice`, поведение не меняется
- [ ] T0.3 `webPresenter` — публикация в SSE-топики `toast`/`confirm` +
      `POST /api/v1/ui/answer`
- [ ] T0.4 Механическая замена `ac.UIService.X` → `ac.Presenter.X`
      (`controller.go`, `elevation.go`, `tray_menu.go`, `ui_watchdog.go`,
      `taildrop*.go`, `services/api_service.go`, `daemon_manager.go`, `rebuild.go`)
- [ ] T0.5 `core/headless.App` — сборка сервисов без Fyne; `--web`,
      `--web-only`, `--listen`, `--tls-cert/--tls-key`, `--web-password-stdin`
- [ ] T0.6 build-теги: `headless` (без Fyne), `noweb` (без статики);
      `tools/win7guard` — зелёный
- [ ] T0.7 Проверка: `go list -deps` для `-tags headless` не содержит `fyne.io/*`
- [ ] T0.8 Десктоп руками: старт, трей, диалоги, taildrop — регрессий нет

## W1 — демон и машины

- [ ] T1.1 `core/lxweb/server.go` + `/api/v1` роутинг + `auth.go`
      (bearer, cookie-сессия, argon2id, rate-limit, `X-LX-Web` на мутациях)
- [ ] T1.2 `/api/v1/daemon/*`: паспорт, статус, start/stop/rollback/apply,
      config, commands (+`POST /commands/{name}`), clients, labels, resources, wire
- [ ] T1.3 `/api/v1/machines/*`: список/строка/здоровье/деплой/ресурсы/
      config active|built|rebuild/backup + `service` (раннбук SPEC 161)
- [ ] T1.4 `events.go` — SSE-мультиплексор: 14 gRPC-стримов демона, топики,
      drop-oldest, `: dropped N`, `?machine=`
- [ ] T1.5 Статика W1: `dash`, `proxies`, `service`, `logs`, `conns`
- [ ] T1.6 Тест `TestLXWebEventMultiplexerDropOldest`
- [ ] T1.7 Живая проверка на роутере (владелец): телефон в LAN, телефон вне LAN

## W2 — состояние

- [ ] T2.1 `core/headless/state_access.go` — вынести из `debugapi`
      (мувтекс, schema-гейт, dirty-маркеры, notify); debugapi переводится на него
- [ ] T2.2 `rev` в состоянии + `If-Match` на мутациях + `409` с диффом
- [ ] T2.3 `/api/v1/state/*`: дерево, nodes, subscriptions (+refresh),
      folders, chains, directions, vars, rules, dns
- [ ] T2.4 Статика W2: `nodes`, `routes`, редакторы правил/DNS/vars
- [ ] T2.5 Тест `TestLXWebStateRevConflict` (+ миграция: старый state без rev)
- [ ] T2.6 Паритет с Fyne-визардом по списку операций (чек-лист в отчёте)

## W3 — реестр как UI

- [ ] T3.1 `/api/v1/registry/{schemes,schemes/{s},warnings,allowlists,vars}`
      на `registry.FormFields`/`FormState` (сервер ничего не считает сам)
- [ ] T3.2 `/api/v1/parse` и `/api/v1/emit` (`linkmap` + тексты warning'ов по локали)
- [ ] T3.3 `fields.js` — рендерер по DTO (все типы: string/enum/bool/int/
      duration/list/range, secret, variants, choices, `implies`, `fixed`)
- [ ] T3.4 Подписи полей: либо R4 (подписи в контракт), либо `tools/labelsgen`
      из `ui/configurator/business/node_form_labels.go` (разово)
- [ ] T3.5 Корпус: `contract/corpus/node_edit` проходит через веб-путь
      без правок ожиданий
- [ ] T3.6 Статика W3: `editor` (блоки-аккордеоны, варианты транспорта,
      REALITY/uTLS, AWG 2.0-блок)

## W4 — сборка и доставка

- [ ] T4.1 `/api/v1/build/preview` (шаблон + vars + ноды → JSON + отчёт)
- [ ] T4.2 `/api/v1/deploy` — ресурсы → конфиг, `422` при отказе демона,
      откат на last-good
- [ ] T4.3 `/api/v1/backup/*` — экспорт/импорт LX Backup 1.0 (dry-run сначала)
- [ ] T4.4 Статика W4: `build` (отчёт по `BuildReportEntry`), `backup`
- [ ] T4.5 Живой цикл: собрал → проверил → задеплоил → увидел в `/chains`

## W5 — безопасность и мобильное

- [ ] T5.1 Режимы `local`/`lan`/`public`, TLS, `trusted_proxies`
- [ ] T5.2 Аудит мутаций (расширение `debugapi/mutation_log.go`: IP, сессия)
- [ ] T5.3 `?redact=1` для скриншотов
- [ ] T5.4 `docs/SECURITY_MODEL.md` — новый раздел «Веб-консоль»
- [ ] T5.5 Локализация UI (en/ru) из `bin/locale` + `warnings.json`
- [ ] T5.6 Мобильная вёрстка: `< 900px` одна колонка, `< 600px` нижняя навигация
- [ ] T5.7 `docs/API.md` — раздел «LX Web API» (второй клиент фасада)
- [ ] T5.8 `docs/ARCHITECTURE.md` §новый — слой фасада и `Presenter`
- [ ] T5.9 `docs/release_notes/upcoming.md` + `RELEASE_NOTES.md`
- [ ] T5.10 CI: джоба `web` (`go build -tags headless`, `go vet` пакета),
      без npm

## Долг, который волны закрывают попутно

- [ ] D1 `core` перестаёт импортировать Fyne (18 файлов) — тестируемость core
- [ ] D2 Версия состояния (`rev`) — закрывает гонку load-modify-save в `/state/*`
- [ ] D3 Подписи полей формы → данные (нужно и LxBox) — бамп контракта
