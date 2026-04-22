---
name: setup
description: "AI Hub initial setup — configure company environment and personal tokens"
argument-hint: "[buildin_config_page_url]"
allowed-tools: ["Bash", "Read", "AskUserQuestion", "mcp"]
---

# AI Hub Setup — первичная настройка окружения

Пошаговый мастер: залогинит в Buildin / Time / Holst через браузер и поможет вставить токен из Kaiten. Агент ведёт диалог — юзер только подтверждает шаги и копирует токен из Kaiten.

## Константы

```
ENV_MANAGER           = integrations/hub-meta/scripts/env-manager.sh
BUILDIN_LOGIN_SCRIPT  = integrations/buildin/scripts/buildin-login.sh
BUILDIN_PAGES_SCRIPT  = integrations/buildin/scripts/buildin-pages.sh
TIME_LOGIN_SCRIPT     = integrations/time/scripts/time-login.sh
DEFAULT_CONFIG_PAGE   = https://buildin.ai/c7ec2023-9025-4c09-be09-e6f54cb07f7e
```

`DEFAULT_CONFIG_PAGE` захардкожен намеренно: после логина в Buildin эта страница отдаёт
`KAITEN_DOMAIN`, `TIME_BASE_URL`, `BUILDIN_SPACE_ID` и т.п. Если URL-аргумент передан — он
перекрывает дефолт.

## Tone

**Говори с юзером дружелюбно, по-русски, от первого лица.** Каждый шаг — одна-две фразы:
«щас откроется Chrome, залогинься через Google SSO и скажи "готово"». Никакого сухого
технического тона. Юзер не должен гадать, что произойдёт дальше — всегда предупреждай
заранее («сейчас открою …», «дальше попрошу токен …»).

## Workflow

### Step 0: Приветствие + миграция

Скажи юзеру коротко и бодро:

> «Сейчас настроим AI Hub. Пройдём 4 шага: логин в Buildin → автоподтянем общий конфиг
> (URL-ы Kaiten и Time) → логин в Time → логин в Holst → токен Kaiten. Минут 5, всё через
> браузер кроме Kaiten-токена — его надо будет разово скопировать из UI. Поехали?»

Если юзер отказался — остановись. Иначе — мигрируй старый `.env.local` (если есть):

```bash
bash integrations/hub-meta/scripts/env-manager.sh migrate
```

- `migrated:*` → скажи «переименовал `.env.local` → `.env`».
- `warning:*` → предупреди, что лежат оба файла, и попроси объединить вручную.
- `ok:nothing to migrate` → молчи.

### Step 0.5: Проверь браузерный MCP

Многие шаги требуют браузерный MCP (Chrome DevTools MCP или Playwright MCP). Проверь
доступность один раз, чтобы не спотыкаться потом.

**Вариант A — Chrome DevTools MCP (приоритет):** вызови read-only тул (`list_pages`).
Если отвечает — используй его.

**Вариант B — Playwright MCP:** вызови `browser_snapshot`. Если отвечает — fallback.

**Вариант C — MCP нет:** скажи юзеру:
> «Нужен браузерный MCP, чтобы я мог открыть окно Chrome и перевести тебя в Buildin/Time.
> Поставлю Chrome DevTools MCP? Без него логин-скиллы и генератор презентаций не заработают.»

С согласия — установи автоматически. Если юзер отказался — предупреди: «Без MCP придётся
вручную копировать токены из DevTools Console. Если передумаешь — просто запусти
`/ai-hub:buildin-login` позже, он сам предложит поставить MCP».

### Step 1: Проверь что уже настроено

```bash
bash integrations/hub-meta/scripts/env-manager.sh check
```

- Если все обязательные `=set` → скажи: «Всё уже настроено! Проверю токены…» и перейди
  к Step 6 (проверить валидность токенов).
- Иначе — кратко перечисли что `missing` (по названиям ключей) и переходи к Step 2.

### Step 2: Логин в Buildin (первый — там лежит общий конфиг)

Скажи юзеру:
> «Сначала Buildin: там на специальной странице лежит общий конфиг команды (URL-ы Kaiten
> и Time). Щас открою Chrome на странице логина — залогинься через Google SSO, а я автоматом
> достану токен (в контекст LLM он не попадёт). Скажи "готово", когда залогинишься.»

Выполни:

```bash
bash integrations/buildin/scripts/buildin-login.sh check
```

- `ok Name (email)` → уже залогинен. Скажи: «О, ты уже залогинен как *Name*, пропускаю.»
- `error:*` → запусти `/ai-hub:buildin-login` (он сам разберётся с MCP и сохранит токен).

После успешного логина → Step 3.

### Step 3: Автоподтягивание конфига из Buildin

Скажи:
> «Читаю страницу с конфигом команды…»

Определи URL конфиг-страницы:

1. Если `$ARGUMENTS` непустой — используй его.
2. Иначе — используй `DEFAULT_CONFIG_PAGE`.

Извлеки `page_id` (последний UUID в URL):

```bash
bash integrations/buildin/scripts/buildin-pages.sh read "<page_id>"
```

Разбери вывод: найди все строки формата `KEY=VALUE`, где KEY — `[A-Z_]+`, VALUE —
непустое. Для каждой пары:

```bash
bash integrations/hub-meta/scripts/env-manager.sh set "<KEY>" "<VALUE>"
```

Покажи юзеру только список записанных **ключей** (значения не выводи):
> «Подтянул: `KAITEN_DOMAIN`, `TIME_BASE_URL`, `BUILDIN_SPACE_ID`.»

**Если страница не открылась (404/403) или в ней нет `KEY=VALUE`:**

Извинись и перейди в ручной режим — спроси у юзера через `AskUserQuestion`:

- `KAITEN_DOMAIN` — например, `yourcompany.kaiten.ru` (без https://)
- `TIME_BASE_URL` — полный URL Time/Mattermost, например, `https://time.yourcompany.io`
- `BUILDIN_SPACE_ID` — ID пространства Buildin (можно пропустить)

Сохрани каждое через `env-manager.sh set`. Если юзер не знает какой-то URL — попроси
открыть соответствующий сервис в браузере и скопировать из адресной строки.

### Step 4: Логин в Time

Скажи:
> «Теперь Time. Откроется вкладка Chrome — залогинься через Google SSO. Дальше мне надо
> будет забрать cookie `MMAUTHTOKEN` из DevTools, я подскажу как.»

Запусти:

```bash
bash integrations/time/scripts/time-login.sh check
```

- `ok @user (email)` → уже залогинен, скажи и пропусти.
- `error:*` → запусти `/ai-hub:time-login` (он откроет Time и проведёт по шагам).

### Step 5: Логин в Holst (опционально)

Holst не хранит токен отдельно — он использует сессию браузера в том же Chrome MCP.
Чтобы `/ai-hub:holst-export` работал без сюрпризов, дай юзеру залогиниться заранее.

Скажи:
> «Теперь Holst (доски, аналог Miro). Токен хранить не надо — просто залогинься в Chrome,
> и сессия останется. Это нужно только если будешь пользоваться `/ai-hub:holst-export`.
> Пропустить?»

Если юзер согласен настроить — через Chrome DevTools MCP `navigate_page` открой
`https://app.holst.so/`. Скажи:
> «Залогинься. Скажи "готово", когда будешь в рабочем пространстве.»

Ничего сохранять не надо — просто подтверди, что вкладка осталась залогиненной.

### Step 6: Токен Kaiten (единственный ручной шаг)

Kaiten API-токен придётся скопировать руками — это осознанное решение: токен даёт полный
доступ к аккаунту, и пусть юзер сам увидит, где он лежит.

Сначала проверь — может уже настроен:

```bash
bash integrations/hub-meta/scripts/env-manager.sh has KAITEN_TOKEN && echo set || echo missing
```

Если `set` — скажи «Kaiten-токен уже есть, проверяю…» и быстро дёрни Kaiten API, например:

```bash
source .env && curl -sS -o /dev/null -w "%{http_code}\n" \
  -H "Authorization: Bearer $KAITEN_TOKEN" "https://$KAITEN_DOMAIN/api/latest/users/current"
```

`200` — ок, пропусти шаг. Не `200` — попроси обновить токен.

Если `missing` — скажи юзеру:

> «Последний шаг — Kaiten-токен. Открой в браузере `https://<KAITEN_DOMAIN>/profile`
> (я подставлю твой домен), потом → **Настройки профиля → API/Интеграции → Создать токен**.
> Скопируй его и пришли мне сюда — я сохраню в `.env`, в контекст LLM токен не уйдёт.»

Подставь актуальный `$KAITEN_DOMAIN` в текст. Когда юзер пришлёт токен:

```bash
bash integrations/hub-meta/scripts/env-manager.sh set KAITEN_TOKEN "<token>"
```

**GENIE_TOKEN** (Databricks) — опционально, только если `GENIE_HOST` настроен. Скажи:
> «Ещё есть Genie (аналитика по DWH) — опционально. Токен выдаёт админ данных. Настроить
> сейчас или пропустить?»

Если настраиваем — аналогично сохрани через `env-manager.sh set GENIE_TOKEN <token>`.

### Step 7: team-config.json — ID досок, колонок, каналов

```bash
test -f team-config.json && echo exists || echo missing
```

Если `missing`:

1. Проверь шаблон: `test -f team-config.example.json` (или в subtree-префиксе, если хаб
   подключён через subtree — например `integrations/sagos95-ai-hub/team-config.example.json`).
   Если шаблон есть — `cp <example> team-config.json`.

2. Скажи:
   > «Создал `team-config.json` из шаблона. Там нужны ID вашей спринтовой доски,
   > бизнес-бэклога, колонок и каналов Time. Можно:
   > — заполнить вручную (открой файл и подставь ID), или
   > — сейчас вместе: я задам вопросы и впишу значения сам. Как удобнее?»

3. Если «вместе» — через `AskUserQuestion` задавай по одному полю (space_id, board_id,
   column IDs и т.д.). Для Kaiten-доски принимай URL — извлеки ID после `/board/`.
   После каждого ответа — `jq` в `team-config.json`:

   ```bash
   tmp=$(mktemp)
   jq '.kaiten.space_id = ($v | tonumber)' --arg v "$ANSWER" team-config.json > "$tmp" \
     && mv "$tmp" team-config.json
   ```

   В конце покажи итог: `jq . team-config.json`.

4. Если «вручную» — просто назови путь и список ключей.

Если `exists` — ничего не делай, упомяни: «`team-config.json` уже есть, если нужно —
отредактируй руками.»

### Step 8: Финальная проверка + хэндофф

```bash
bash integrations/hub-meta/scripts/env-manager.sh check
```

Покажи итог. Если остались `=missing` в optional — скажи, что это ок и настроить можно
позже.

Заверши позитивно:

> «Готово! Попробуй что-нибудь:
> • `/ai-hub:time-chat read <channel>` — прочитать канал в Time
> • `/ai-hub:buildin-read <url>` — открыть страницу Buildin
> • `/ai-hub:spike <kaiten-url>` — полный spike по карточке
> Если что-то сломается — запусти `/ai-hub:setup` ещё раз, он идемпотентный.»

## Обработка ошибок

- **Buildin login не прошёл** → предложи ручной fallback из `/ai-hub:buildin-login`
  (скопировать JWT из DevTools Console). Если совсем никак — перейди на manual ввод
  `KAITEN_DOMAIN` / `TIME_BASE_URL` в Step 3.
- **Страница конфига пустая или 404** → извинись, перейди в manual-режим (Step 3 fallback).
- **`env-manager.sh set` вернул ошибку** → покажи ошибку, предложи отредактировать `.env`
  вручную.
- **Kaiten токен не валиден (403/401)** → попроси пересоздать токен и повторить Step 6.
