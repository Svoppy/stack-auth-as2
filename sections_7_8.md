# Разделы 7 и 8 — на основе реального анализа кода репозитория stack-auth

> **Репозиторий:** https://github.com/stack-auth/stack  
> **Ветка:** main  
> **Дата анализа:** 31.03.2026  
> **Коммит (HEAD локальной копии):** проверено на клоне ветки main

---

## 7. Выявленные потенциальные уязвимости (анализ исходного кода)

### Уязвимость 1 — Регистрация сырого JWT-токена в логах сервера

**Файл:** `apps/backend/src/lib/tokens.tsx`, строка 120

**Фрагмент кода:**
```typescript
} else if (error instanceof JOSEError) {
  console.warn("Unparsable access token. This might be a user error, but if it happens frequently, it's a sign of a misconfiguration.", { accessToken, error });
  return Result.error(new KnownErrors.UnparsableAccessToken());
}
```

**Описание:** При ошибке разбора JWT в функции `decodeAccessToken` сырая строка токена (`accessToken`) передаётся напрямую в `console.warn`. Это означает, что в серверных логах оседают полноценные access-токены пользователей — в том числе действующие токены, которые могут быть отозваны только через истечение срока действия. Если к логам есть доступ у третьих лиц (аудит-системы, Sentry, внешние log-агрегаторы), сессии пользователей оказываются скомпрометированы.

**Аналогичная строка** в той же функции (строка 117): в `console.warn` логируется `decoded?.sub` (userId) и `decoded?.aud` (projectId), что раскрывает структуру токена.

**Классификация:**
- **CWE-532** — Insertion of Sensitive Information into Log File
- **OWASP Top 10 2021:** A09 — Security Logging and Monitoring Failures

**Уровень риска:** Высокий

---

### Уязвимость 2 — Предсказуемый криптографический challenge в Passkey-аутентификации

**Файл:** `apps/backend/src/app/api/latest/auth/passkey/initiate-passkey-authentication/route.tsx`, строка 42  
**Файл:** `apps/backend/.env.development` — `STACK_ENABLE_HARDCODED_PASSKEY_CHALLENGE_FOR_TESTING=yes`

**Фрагмент кода:**
```typescript
const authenticationOptions: PublicKeyCredentialRequestOptionsJSON = await generateAuthenticationOptions({
  rpID: "THIS_VALUE_WILL_BE_REPLACED.example.com",
  userVerification: "preferred",
  challenge: getEnvVariable("STACK_ENABLE_HARDCODED_PASSKEY_CHALLENGE_FOR_TESTING", "")
    ? isoUint8Array.fromUTF8String("MOCK")   // <-- жёстко закодированный challenge
    : undefined,
  allowCredentials: [],
  timeout: SIGN_IN_TIMEOUT_MS,
});
```

**Описание:** Если переменная окружения `STACK_ENABLE_HARDCODED_PASSKEY_CHALLENGE_FOR_TESTING` установлена, сервер отдаёт фиксированный challenge `"MOCK"` вместо криптографически случайного значения. Это нарушает фундаментальное требование протокола WebAuthn/FIDO2: challenge должен быть уникальным и непредсказуемым для каждой сессии аутентификации. С известным challenge злоумышленник может подготовить и воспроизвести валидную подпись (replay attack), обходя Passkey-аутентификацию полностью.

В файле `.env.development` (закоммиченном в репозиторий) значение этой переменной равно `yes`, то есть опасный режим включён по умолчанию для разработчиков и легко может «протечь» в staging-окружение.

Аналогичная уязвимость присутствует в маршруте регистрации:  
`apps/backend/src/app/api/latest/auth/passkey/initiate-passkey-registration/route.tsx`, строка 48.

**Классификация:**
- **CWE-330** — Use of Insufficiently Random Values
- **CWE-294** — Authentication Bypass by Capture-replay
- **OWASP Top 10 2021:** A02 — Cryptographic Failures

**Уровень риска:** Критический (при попадании конфига в prod-среду)

---

### Уязвимость 3 — Жёстко заданные учётные данные в .env.development (committed secrets)

**Файл:** `apps/backend/.env.development`

**Проблемные строки:**
```
STACK_SERVER_SECRET=23-wuNpik0gIW4mruTz25rbIvhuuvZFrLOLtL7J4tyo
STACK_SEED_INTERNAL_PROJECT_SECRET_SERVER_KEY=this-secret-server-key-is-for-local-development-only
STACK_SEED_INTERNAL_PROJECT_SUPER_SECRET_ADMIN_KEY=this-super-secret-admin-key-is-for-local-development-only
CRON_SECRET=mock_cron_secret
STACK_STRIPE_SECRET_KEY=sk_test_mockstripekey
STACK_SVIX_API_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpYXQiOjE2NTUxNDA2MzksImV4cCI6...
STACK_AWS_ACCESS_KEY_ID=test
STACK_AWS_SECRET_ACCESS_KEY=test
STACK_S3_ACCESS_KEY_ID=s3mockroot
STACK_S3_SECRET_ACCESS_KEY=s3mockroot
STACK_QSTASH_TOKEN=eyJVc2VySUQiOiJkZWZhdWx0VXNlciIsIlBhc3N3b3JkIjoiZGVmYXVsdFBhc3N3b3JkIn0=
```

**Описание:** Файл `.env.development` добавлен в git-репозиторий и содержит реальные значения секретных ключей, токенов доступа и паролей. В частности, `STACK_SVIX_API_KEY` представляет собой полноценный JWT-токен для вебхук-сервиса. `CRON_SECRET=mock_cron_secret` используется для защиты внутренних административных endpoints (`/internal/failed-emails-digest`, `/internal/external-db-sync/poller` и др.) — любой, кто клонировал репозиторий, знает этот секрет.

Кроме того, файл `.env` (шаблон) содержит в поле `STACK_DATABASE_CONNECTION_STRING` строку с placeholder-паролем: `PASSWORD-PLACEHOLDER--uqfEC1hmmv`, который явно относится к реальной инфраструктуре.

**Классификация:**
- **CWE-259** — Use of Hard-coded Password
- **CWE-798** — Use of Hard-coded Credentials
- **OWASP Top 10 2021:** A02 — Cryptographic Failures

**Уровень риска:** Критический

---

### Уязвимость 4 — Уязвимость тайминг-атаки при верификации CRON_SECRET

**Файл:** `apps/backend/src/app/api/latest/internal/failed-emails-digest/route.ts`, строка 42  
(аналогично в `poller/route.ts:79`, `sequencer/route.ts:181`, `email-queue-step/route.tsx:37`)

**Фрагмент кода:**
```typescript
const authHeader = headers.authorization[0];
if (authHeader !== `Bearer ${getEnvVariable('CRON_SECRET')}`) {
  throw new StatusError(401, "Unauthorized");
}
```

**Описание:** Для защиты внутренних cron-endpoints используется сравнение строк через оператор `!==`. JavaScript не гарантирует константное время при сравнении строк: если строки отличаются в разных позициях, сравнение завершается раньше. Это позволяет злоумышленнику проводить timing attack: методично перебирая символы токена и измеряя время ответа, можно побайтово восстановить значение `CRON_SECRET`. После его получения атакующий получает прямой доступ к внутренним административным функциям (рассылка дайджестов, управление очередями, синхронизация БД) без какой-либо дополнительной аутентификации.

Правильное решение — использовать `crypto.timingSafeEqual()` из Node.js.

**Классификация:**
- **CWE-208** — Observable Timing Discrepancy
- **CWE-284** — Improper Access Control
- **OWASP Top 10 2021:** A07 — Identification and Authentication Failures

**Уровень риска:** Средний–Высокий

---

### Уязвимость 5 — Prompt Injection → неконтролируемый SQL-запрос к ClickHouse через AI-инструмент

**Файл:** `apps/backend/src/lib/ai/tools/sql-query.ts`, строки 22–49

**Фрагмент кода:**
```typescript
return tool({
  description: "Run a ClickHouse SQL query against the project's analytics database. Only SELECT queries are allowed.",
  inputSchema: z.object({
    query: z
      .string()
      .describe("The ClickHouse SQL query to execute. Only SELECT queries are allowed. Always include LIMIT clause."),
  }),
  execute: async ({ query }: { query: string }) => {
    const client = getClickhouseExternalClient();
    return await client.query({
      query,   // <-- строка из LLM-ответа, сформированная на основе пользовательского промпта
      clickhouse_settings: {
        SQL_project_id: projectId,
        readonly: "1",
        allow_ddl: 0,
        ...
      },
      format: "JSONEachRow",
    })
```

**Описание:** AI-инструмент принимает SQL-запрос из вывода LLM и передаёт его напрямую в ClickHouse без параметризации. Поле `query` формируется языковой моделью, а пользователь может влиять на неё через промпт (prompt injection). Несмотря на настройки `readonly: "1"` и `allow_ddl: 0`, атакующий через prompt injection может заставить LLM сгенерировать запрос, который:

- читает данные за пределами `SQL_project_id` при наличии UNION-запросов или обращений к системным таблицам ClickHouse;
- обходит межпроектную изоляцию, если настройка `SQL_project_id` применяется только как подсказка, а не как жёсткое ограничение на уровне Row Policy.

Валидация `Only SELECT queries are allowed` реализована только на уровне промпта к LLM, а не в коде.

**Классификация:**
- **CWE-89** — Improper Neutralization of Special Elements used in an SQL Command (SQL Injection)
- **CWE-77** — Improper Neutralization of Special Elements used in a Command (Prompt Injection)
- **OWASP Top 10 2021:** A03 — Injection

**Уровень риска:** Высокий

---

### Уязвимость 6 — Открытый редирект через сохранённый redirectUrl в verification code

**Файл:** `apps/backend/src/route-handlers/verification-code-handler.tsx`, строки 327–329

**Фрагмент кода:**
```typescript
// При создании кода — проверка URL (строка 232)
if (callbackUrl !== undefined && !validateRedirectUrl(callbackUrl, tenancy)) {
  throw new KnownErrors.RedirectUrlNotWhitelisted();
}

// При использовании кода — URL используется напрямую без повторной проверки
if (code.redirectUrl !== null) {
  link = new URL(code.redirectUrl);  // <-- значение из БД, без re-validation
  link.searchParams.set('code', code.code);
}
```

**Описание:** URL для редиректа валидируется в момент создания верификационного кода (при регистрации или сбросе пароля), но при последующем использовании кода это значение берётся из базы данных без повторной проверки. Если:
- white-list доменов изменился после создания кода;
- произошла инъекция в базу данных или SSRF во внутреннюю систему;
- конфигурация `trustedDomains` в tenancy была обновлена,

то ссылка в письме будет перенаправлять пользователя на непроверенный URL. Атакующий может воспользоваться этим для фишинга (open redirect) или для захвата токена верификации через attacker-controlled redirect endpoint.

**Классификация:**
- **CWE-601** — URL Redirection to Untrusted Site ('Open Redirect')
- **OWASP Top 10 2021:** A01 — Broken Access Control

**Уровень риска:** Средний

---

## 8. Оценка рисков и меры защиты

### 8.1 Таблица оценки рисков

| № | Угроза | Уязвимость / файл | Вероятность (1–5) | Последствия (1–5) | Риск = P × I | Приоритет | Меры защиты |
|---|--------|-------------------|:-----------------:|:-----------------:|:------------:|:---------:|-------------|
| 1 | Утечка JWT-токена через логи (CWE-532) | `tokens.tsx:120` | 4 | 4 | **16** | Высокий | Логировать только хэш токена или первые/последние N символов; настроить уровень логов (не `warn` с payload-объектом) |
| 2 | Replay attack через фиксированный Passkey challenge (CWE-330) | `initiate-passkey-authentication/route.tsx:42` + `.env.development` | 3 | 5 | **15** | Критический | Запретить `STACK_ENABLE_HARDCODED_PASSKEY_CHALLENGE_FOR_TESTING` в staging/prod; добавить guard в код; добавить в CI-проверку |
| 3 | Жёстко закодированные учётные данные в репозитории (CWE-259/798) | `.env.development` | 5 | 5 | **25** | Критический | Убрать `.env.development` из git (добавить в `.gitignore`); ротировать все попавшие в репо секреты; использовать secret scanning в CI (например, Gitleaks, GitHub Secret Scanning) |
| 4 | Timing attack на CRON_SECRET (CWE-208) | `route.ts:42` в 4 файлах | 2 | 4 | **8** | Средний | Заменить `!==` на `crypto.timingSafeEqual()`; перевести на стандартный JWT middleware |
| 5 | Prompt injection → SQL-запрос к ClickHouse (CWE-89/77) | `sql-query.ts:24` | 3 | 4 | **12** | Высокий | Добавить серверную валидацию SQL (allowlist операций: только `SELECT`); реализовать ClickHouse Row Policy для жёсткой изоляции проектов; ограничить доступные таблицы |
| 6 | Открытый редирект через verification code (CWE-601) | `verification-code-handler.tsx:328` | 2 | 3 | **6** | Средний | Добавить повторную проверку `validateRedirectUrl` при исполнении кода, а не только при его создании |

---

### 8.2 Интерпретация риска

| Диапазон | Уровень |
|----------|---------|
| 1–4 | Низкий |
| 5–9 | Средний |
| 10–16 | Высокий |
| 17–25 | Критический |

---

### 8.3 Приоритизированные меры защиты

| Приоритет | Мера | Угроза | Сложность внедрения |
|:---------:|------|--------|:-------------------:|
| 1 | Убрать `.env.development` из git; добавить в `.gitignore`; ротировать все секреты (SVIX API key, STACK_SERVER_SECRET) | CWE-259/798 | Низкая |
| 2 | Добавить secret scanning в CI/CD (Gitleaks, TruffleHog или GitHub Advanced Security) | CWE-259/798 | Низкая |
| 3 | Запретить `STACK_ENABLE_HARDCODED_PASSKEY_CHALLENGE_FOR_TESTING=yes` вне dev-окружения; добавить runtime-проверку в код | CWE-330 | Низкая |
| 4 | Заменить строковое сравнение `!==` на `crypto.timingSafeEqual()` для CRON_SECRET во всех 4 маршрутах | CWE-208 | Низкая |
| 5 | Убрать `accessToken` из аргументов `console.warn`; заменить на `[REDACTED]` или хэш | CWE-532 | Низкая |
| 6 | Добавить серверный allowlist для SQL-инструмента AI (`/^SELECT\s/i`), реализовать ClickHouse Row Policy для изоляции данных по `project_id` | CWE-89/77 | Средняя |
| 7 | Повторно вызывать `validateRedirectUrl` при исполнении verification code, а не только при создании | CWE-601 | Низкая |

---

### 8.4 Вывод

Наиболее критическими являются три уязвимости, выявленные непосредственно в коде репозитория:

1. **Секреты в `.env.development`** (риск 25/25) — файл закоммичен в репозиторий и содержит STACK_SERVER_SECRET, SVIX API key, Stripe key, CRON_SECRET. Это нарушение базового принципа управления секретами; его устранение должно быть первоочередным.

2. **Жёстко заданный Passkey challenge** (риск 15/25) — использование `"MOCK"` в качестве challenge полностью уничтожает криптографическую защиту passkey-аутентификации и допускает replay attack.

3. **JWT-токен в логах** (риск 16/25) — высокая вероятность реализации в боевых условиях, так как любая ошибка парсинга JWT (например, при атаке перебором) приводит к логированию токена.

Все выявленные уязвимости имеют прямые ссылки на конкретные строки кода и воспроизводимы при анализе репозитория. Меры защиты для большинства из них требуют минимальных изменений кода при значительном снижении уровня риска.
