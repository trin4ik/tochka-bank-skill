# Скилл для Точка Банк API

Скилл для AI-агентов для работы с REST API Точка.Банк. Работает с **любым агентом, поддерживающим стандарт skills** — Claude Code, Cursor, Codex, Gemini CLI, Goose, Windsurf, Roo Code, Cline и [40+ других агентов](https://skills.sh).

## Что умеет

Python CLI (только стандартная библиотека) + структурированные знания для работы с REST API Точка.Банк (developers.tochka.com).

### Возможности

- **Счета на оплату** — создание, автозагрузка PDF, отправка на email, проверка статуса оплаты, удаление
- **Закрывающие документы** — акт выполненных работ, УПД, ТОРГ-12, счёт-фактура
- **Полная выписка** через Open Banking (асинхронный flow с polling)
- **Входящие платежи** — СБП и карты через эквайринг
- **Платёжные поручения** — создание черновиков «На подпись»
- **Платёжные ссылки** для интернет-эквайринга (с two-stage capture, ОФД-чеками)
- **Вебхуки** — регистрация HTTPS-endpoint'а для push-уведомлений, тестовая отправка
- **СБП QR-коды** — регистрация мерчанта, статический / динамический / cash QR
- **Реестр эквайринга** — суточная сверка комиссий
- **Диагностика OAuth** — introspection согласий (consents) при 403

### Особенности реализации

- **Два потока аутентификации** — личный JWT (просто, для чтения + черновиков платёжек) или OAuth 2.0 + Consent (для Invoice API, закрывающих документов и полной выписки)
- **Безопасное хранение секретов** — macOS Keychain / Linux secret-tool / Windows Credential Manager, файловый fallback; access_token OAuth автоматически обновляется
- **Интерактивные визарды** `init` и `init --oauth` с проверкой TTY, подсказками по регистрации приложения и локальным HTTPS-callback сервером (поддерживает mkcert)
- **Флаг `--format {json,id,url}`** у команд `create-*` — получить только ID или URL в stdout для shell-пайплайнов, без `jq`
- **Опциональный хук подтверждения для Claude Code** — permission-prompt с метками `[PROD]` / `[SANDBOX]` перед каждым изменяющим вызовом
- **Offline OpenAPI-спека** (OpenAPI 3.1.0, Tochka.API v1.90.4-stable, 440 КБ в `schemas/`) для поиска схем через `jq`

### Референсная библиотека

| Файл | Содержание |
|------|-----------|
| `SKILL.md` | Роутер для агента: Quickstart (выбор по цели), матрица JWT vs OAuth, таблица задача → команда, критичные ошибки схемы, шпаргалка error → fix (18 строк) |
| `references/auth.md` | JWT и OAuth flows, полный список 19 разрешений, устройство визардов, подводные камни регистрации приложения (включая баг `localhost` vs `127.0.0.1`) |
| `references/endpoints.md` | Схемы всех 19+ эндпоинтов, правила валидации, различия полей для ИП vs ООО |
| `references/webhooks.md` | Верификация подписи через OIDC discovery, retry semantics, идемпотентность |
| `references/errors.md` | Шпаргалка error → fix (HTTP 501/403/401, OAuth callback, Keychain, PDF render) |
| `schemas/swagger.json` | Offline OpenAPI 3.1.0 спека (440K). Источник: `https://enter.tochka.com/doc/openapi/swagger.json`. Читать через `jq`, не загружать в контекст. |

## Установка

### Через Skills CLI (рекомендуется)

```bash
npx skills add rodion-m/tochka-bank-skill@tochka-bank-api -g -y
```

### Вручную

Скопируйте директорию `tochka-bank-api/` (с `SKILL.md`, `references/`, `schemas/`, `scripts/`, `hooks/`) в папку скиллов вашего агента:

| Агент | Путь |
|-------|------|
| Claude Code | `~/.claude/skills/tochka-bank-api/` |
| Cursor | `~/.cursor/skills/tochka-bank-api/` |
| Codex | `~/.codex/skills/tochka-bank-api/` |
| Gemini CLI | `~/.gemini/skills/tochka-bank-api/` |
| Windsurf | `~/.codeium/windsurf/skills/tochka-bank-api/` |
| Goose | `~/.config/goose/skills/tochka-bank-api/` |
| Roo Code | `~/.roo/skills/tochka-bank-api/` |

Полный список 42 поддерживаемых агентов — [skills.sh](https://skills.sh).

### Первоначальная настройка

После установки однократно запустите визард в своём терминале. Префикс `!` обязателен — визард читает секреты через `getpass`, которому нужен настоящий TTY (Bash-инструмент самого агента TTY не предоставляет):

```bash
# Просто: личный JWT из ЛК Точки. Для чтения + черновиков платёжек.
# НЕ покрывает счета и выписки (возвращают 501 с личным JWT).
! python3 ~/.claude/skills/tochka-bank-api/scripts/tochka_client.py init

# Полный: OAuth 2.0 + Consent. Для счетов, закрывающих документов, полной выписки.
# Требует однократной регистрации приложения в ЛК Точки.
! python3 ~/.claude/skills/tochka-bank-api/scripts/tochka_client.py init --oauth
```

OAuth-режим требует однократной регистрации приложения на https://i.tochka.com/bank/services/m/integration/new.

## Использование

Скилл активируется автоматически, когда вы просите агента:

- «выставь счёт через Точку»
- «получи выписку из Точки за последнюю неделю»
- «зарегистрируй вебхук на incomingPayment»
- «какие платёжки ждут подписания в Точке»
- «проверь баланс»
- «создай закрывающий акт по счёту N»

Или вызвать напрямую: `/tochka-bank-api` (Claude Code).

## Хук подтверждения (только Claude Code)

В директории скилла есть опциональный хук `hooks/tochka-require-confirmation.sh`, который перехватывает изменяющие вызовы (`curl -X POST|PUT|PATCH|DELETE`, `create-invoice`, `register-webhook` и др.) и требует подтверждения через permission-prompt Claude Code. Метка в запросе показывает целевую среду: `[PROD] create-invoice — реальные деньги / банковские записи могут быть затронуты` vs `[SANDBOX] create-invoice — только песочница, безопасно разрешить`.

Установка:

```bash
mkdir -p .claude/hooks
cp ~/.claude/skills/tochka-bank-api/hooks/tochka-require-confirmation.sh .claude/hooks/
chmod +x .claude/hooks/tochka-require-confirmation.sh
```

Зарегистрируйте в `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/tochka-require-confirmation.sh",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

Механизм хуков специфичен для Claude Code — другие агенты его игнорируют. Read-only команды (`list-*`, `get-*`, `config`, валидация `init`) проходят без запроса.

## Ограничения

- **Только Точка.Банк.** У Тинькофф/Сбер/ВТБ другие схемы — не переносите примеры.
- **Не для мультитенантного SaaS.** OAuth режим поддерживает мультитенантность технически, но визарды оптимизированы под автоматизацию «своего» аккаунта.
- **Отличие от стандартов АФТ/ОБР.** API Точки не соответствует утверждённым ЦБ стандартам АФТ (wiki.openbankingrussia.ru, Accounts v2.0/v3.0, ФАПИ Advanced v2.0). Ожидайте смену схем около **01.10.2026**.
- Некоторые пути и имена полей меняются без строгого semver — всегда сверяйтесь с живым [ReDoc](https://enter.tochka.com/doc/v2/redoc) перед деплоем.

## Вклад в проект

PR приветствуются. Особенно полезны:

- Новые строки в шпаргалку error → fix (точная строка ошибки + причина + исправление).
- Обновления схем при breaking changes Точки.
- Верификация СБП-операций вживую — в скилле нет live-проверенных СБП-вызовов (схемы из swagger + ReDoc).

## Лицензия

[MIT](LICENSE)
