# ПРАКТИЧЕСКАЯ РАБОТА №1

## Моделирование угроз и составление карты поверхности атаки

**Вариант 1**

**CSE-2506**

Выполнил: Казиханов Диас

Проверил: Лисневский Р. В.

---

## 1. Исходные данные

Stack Auth --- открытая платформа аутентификации, альтернатива Auth0 и
Clerk. На GitHub у проекта больше 6 700 звёзд и около 500 форков, то
есть кодовая база используется реально.

Проект устроен как монорепозиторий на Turbo. Внутри четыре основных
части:

**Backend (apps/backend)** --- Next.js-сервер, принимает все
API-запросы. Маршруты лежат в apps/backend/src/app/api/latest, порт
8102.

**Dashboard (apps/dashboard)** --- админка: управление пользователями,
проектами, настройками. Порт 8101.

**Packages (packages/)** --- клиентские SDK для React и Next.js. Здесь
компоненты логина, регистрации, профиля.

**SDKs (sdks/)** --- дополнительные SDK, в том числе экспериментальный
для Swift.

**Стек:** TypeScript, Next.js (серверные компоненты + route handlers),
Prisma ORM, PostgreSQL в Docker, JWT (access + refresh), OAuth 2.0,
magic links, парольная аутентификация. Зависимости через pnpm.

**Пользователи:** разработчики, которые встраивают Stack Auth в свои
приложения; конечные пользователи этих приложений; администраторы через
dashboard.

**Что ценно:** email и хеши паролей, JWT-токены, серверные секреты
(STACK\_SECRET\_SERVER\_KEY), OAuth-конфиги (client\_id/client\_secret),
данные сессий, профили, API-ключи проектов.

---

## 2. Область анализа

Анализировались:

**Backend API** --- эндпоинты аутентификации, авторизации, управления
сессиями, токенами, организациями.

**Dashboard** --- админ-интерфейс, формы ввода, рендеринг
пользовательских данных.

**Клиентские SDK** --- React-компоненты, хуки, работа с токенами на
клиенте.

**Конфиги** --- .env.development, docker-compose, turbo.json,
prisma/schema.prisma.

**Зависимости** --- package.json всех пакетов, pnpm-lock.yaml.

**CI/CD** --- GitHub Actions (.github/workflows).

**Переменные окружения** --- NEXT\_PUBLIC\_STACK\_PROJECT\_ID,
NEXT\_PUBLIC\_STACK\_PUBLISHABLE\_CLIENT\_KEY, STACK\_SECRET\_SERVER\_KEY.

---

## 3. Критичные активы и нарушители

### 3.1. Активы

Учётные записи --- email, хеши паролей, OAuth-привязки. Потеря ---
полный захват аккаунта.

JWT-токены --- access (короткоживущий) и refresh (долгоживущий). Утечка
refresh-токена даёт длительный доступ без пароля.

Серверные секреты --- STACK\_SECRET\_SERVER\_KEY, ключ подписи JWT, OAuth
client\_secret. Если утекут --- атакующий получит полный контроль над
аутентификацией.

Cookies сессий --- управляются Stack Auth, привязаны к пользователю.

Профильные данные --- имя, email, аватар, метаданные организаций.

Конфигурация проектов --- API-ключи (publishable/secret), настройки
OAuth, разрешённые домены.

### 3.2. Нарушители

**Внешние:**

Анонимный атакующий --- без аккаунта. Пробует подобрать пароли,
эксплуатировать баги форм входа/регистрации, перехватить токены.

Целевой атакующий (конкурент) --- есть ресурсы и время. Цель ---
массовая утечка данных или нарушение работы сервиса.

Бот --- credential stuffing, массовая регистрация фейковых аккаунтов.

**Внутренние:**

Скомпрометированный сотрудник --- имеет доступ к репо и инфраструктуре.
Может внедрить бэкдор или вытащить ключи.

Инсайдер --- доступ к dashboard, пытается выйти за рамки своих прав.

---

## 4. Доверительные границы

В архитектуре Stack Auth четыре ключевых границы доверия:

**Граница 1: Браузер --- Frontend.** Браузер --- недоверенная среда.
Всё, что приходит от клиента (формы, параметры), считается потенциально
вредоносным. Клиентская валидация --- только для UX.

**Граница 2: Frontend --- Backend API.** Связь через HTTPS. Backend
проверяет JWT, CORS, CSRF. Publishable key виден всем (NEXT\_PUBLIC\_\*),
а secret key --- только на сервере.

**Граница 3: Backend --- PostgreSQL.** Prisma ORM параметризует запросы.
БД в отдельном Docker-контейнере, доступ только с сервера.

**Граница 4: Backend --- OAuth-провайдеры.** Backend обменивается с
Google, GitHub и т.д. через client\_id/client\_secret. Ответы
верифицируются, но зависимость от внешних сервисов --- отдельный вектор.

Доверенная зона: серверные процессы, БД, переменные окружения.
Недоверенная: браузер, сеть, внешние провайдеры, npm-зависимости.

---

## 5. Модель угроз STRIDE

| Компонент | Категория | Угроза | Источник | Последствия | Риск |
|---|---|---|---|---|---|
| Форма входа (/password/sign-in) | Spoofing | Перебор паролей (credential stuffing) через автоматические запросы | Внешний | Захват чужих аккаунтов | Высокий |
| JWT access-токен | Tampering | Подделка payload при слабой подписи или утечке ключа | Внешний | Эскалация привилегий | Высокий |
| Профиль (/profile) | Info Disclosure | IDOR --- получение данных чужого профиля через перебор ID | Внешний | Утечка email и имён | Средний |
| send-reset-code / reset | Repudiation | Нет логов запросов сброса --- нельзя восстановить картину инцидента | Внешний | Невозможность расследования | Средний |
| Серверные логи | Info Disclosure | Токены или пароли попадают в логи открытым текстом | Внутренний | Извлечение секретов из логов | Высокий |
| PostgreSQL | DoS | Нет rate limiting --- массовые запросы кладут базу | Внешний | Отказ в обслуживании | Средний |
| Dashboard | Elevation of Privilege | Прямые API-вызовы в обход интерфейса дают админ-доступ | Внешний | Контроль чужих проектов | Критический |
| OAuth callback | Spoofing | Подмена redirect\_uri для перехвата authorization code | Внешний | Вход под чужим аккаунтом | Высокий |

---

## 6. Карта поверхности атаки

| Точка входа | Тип данных | Компонент | Доверенность | Куда идут данные | Защита | Угрозы |
|---|---|---|---|---|---|---|
| /api/latest/auth/password/sign-in | email, пароль | Backend API | Нет | Prisma → PostgreSQL | HTTPS, валидация | Credential stuffing, SQLi |
| /api/latest/auth/password/sign-up | email, пароль, имя | Backend API | Нет | Prisma → PostgreSQL | Валидация схемы | Массовая регистрация, слабые пароли |
| /api/latest/auth/password/send-reset-code | email | Backend API | Нет | Токен → SMTP | Rate limiting | Перебор email, enumeration |
| /api/latest/users/me | JSON (имя, аватар) | Backend API | Частично (JWT) | Prisma, БД, HTML | JWT-проверка | XSS, IDOR |
| Cookies (refresh) | Токен | Браузер/Backend | Нет | HTTP headers | HttpOnly, Secure, SameSite | Кража через XSS, CSRF |
| JWT (Authorization) | Access token | Backend API | Частично | Верификация подписи | HMAC/RSA | Replay, утечка ключа |
| URL (redirect\_uri, code) | Строки | OAuth handler | Нет | Обмен code на token | Whitelist URI | Open redirect, CSRF |
| .env / env vars | Секреты | Сервер | Да | Внутреннее | .gitignore | Утечка в CI/CD |
| Webhook endpoints | JSON | Backend API | Внешний сервис | Обработка событий | HMAC-подпись | Подделка запросов |
| package.json / lockfile | Зависимости | Сборка | npm registry | node\_modules | lockfile, audit | Supply chain, typosquatting |

---

## 7. Анализ потоков данных

### 7.1. Логин и пароль → SQL-запрос

**Source:** пользователь вводит email и пароль в форму (React-компонент SignIn), формируется JSON.

**Propagation:** POST /api/latest/auth/password/sign-in
(`apps/backend/src/app/api/latest/auth/password/sign-in/route.tsx`).
Backend разбирает тело через yup-схему (emailSchema, passwordSchema).

**Sanitization:** Prisma параметризует запросы автоматически.
Дополнительно --- zod-валидация. Но если кто-то напишет raw query через
`$queryRawUnsafe` со строковой конкатенацией --- защита не сработает.

**Sink:** `comparePassword` (строка 54) сравнивает пароль с хешем из БД
с защитой от timing-атак. При успехе --- `createAuthTokens` (строки
71--74) создаёт refresh + access JWT-токены (`tokens.tsx:374--382`).

Фрагмент реального кода (`sign-in/route.tsx:54--71`):

```typescript
// sign-in/route.tsx:54-55 — timing-safe сравнение:
if (!await comparePassword(password, passwordAuthMethod?.passwordHash || "")) {
  throw new KnownErrors.EmailPasswordMismatch();
}

// sign-in/route.tsx:71 — генерация токенов:
const { refreshToken, accessToken } = await createAuthTokens({
  tenancy, projectUserId: contactChannel.projectUser.projectUserId,
});
```

---

### 7.2. Поле профиля → HTML (XSS)

**Source:** пользователь заполняет display name через форму
редактирования профиля.

**Propagation:** PATCH /api/latest/users/me
(`apps/backend/src/app/api/latest/users/me/route.tsx`). Данные
сохраняются в БД, потом отображаются в dashboard и компонентах SDK.

**Sanitization:** React экранирует строки в JSX. Но в
`apps/dashboard/src/app/layout.tsx` (строки 82--89 и 92) и
`apps/dashboard/src/components/ui/chart.tsx` (строки 89--108) реально
используется `dangerouslySetInnerHTML` --- места риска.

**Sink:** DOM браузера другого пользователя или админа. Stored XSS ---
скрипт работает в чужой сессии.

Фрагмент реального кода (`layout.tsx:82--92`):

```typescript
// layout.tsx:82-89 — реальный dangerouslySetInnerHTML:
{headTags.map((tag, index) => {
  const { tagName, attributes, innerHTML } = tag;
  return React.createElement(tagName, {
    dangerouslySetInnerHTML: { __html: innerHTML ?? "" },
    ...attributes,
  });
})}

// layout.tsx:92 — inline-скрипт темизации:
<script dangerouslySetInnerHTML={{ __html: "(function(){...})()" }} />
```

---

### 7.3. Email для сброса пароля → токен и письмо

**Source:** email в форме сброса пароля.

**Propagation:** POST /api/latest/auth/password/send-reset-code ---
отправляет письмо; POST /api/latest/auth/password/reset --- применяет
код. Одинаковый ответ независимо от наличия пользователя.

**Sanitization:** email валидируется через yup emailSchema. Защита от
enumeration: одинаковый ответ `{"success": "maybe, only if user with
e-mail exists"}` в обоих ветках (`send-reset-code/route.tsx` строки
50--57 и 77--82).

**Sink:** verification code сохраняется в БД, письмо уходит через
`sendEmailFromDefaultTemplate` (`verification-code-handler.tsx:38--49`).
В dev-режиме --- Inbucket (`apps/backend/.env.development`). Пароль
обновляется через `adminUpdate`.

Фрагмент реального кода (`send-reset-code/route.tsx:50--57`):

```typescript
// send-reset-code/route.tsx:50-56 — anti-enumeration:
if (!contactChannel) {
  await wait(2000 + Math.random() * 1000);
  return {
    statusCode: 200,
    body: { success: "maybe, only if user with e-mail exists" }
  };
}
// Строки 77-81: тот же ответ при успехе — email enumeration невозможен
```

---

## 8. Выявленные уязвимости

### 8.1. CWE-89: Небезопасный raw SQL-запрос ($queryRawUnsafe)

**Где:** `apps/backend/src/lib/external-db-sync.ts`, строка 564.

**Суть:** функция `syncExternalDb` использует `$queryRawUnsafe` --- метод
Prisma, который передаёт SQL без ORM-параметризации. Строка `fetchQuery`
формируется внешним маппингом (`getInternalDbFetchQuery`). Частичная
защита --- проверка наличия плейсхолдеров `$1` и `$2` (строки 531--535),
но она не гарантирует, что структура запроса не содержит других
опасных конструкций. При некорректном маппинге возможна SQL-инъекция.

Реальный код (`external-db-sync.ts:531--564`):

```typescript
// Строки 531-535 — проверка плейсхолдеров:
if (!fetchQuery.includes("$1") || !fetchQuery.includes("$2")) {
  throw new StackAssertionError(
    `internalDbFetchQuery must reference $1 (tenancyId) and $2 (lastSequenceId).`
  );
}

// Строка 564 — реальный вызов $queryRawUnsafe:
const rows = await internalPrisma.$replica().$queryRawUnsafe<any[]>(
  fetchQuery, tenancyId, lastSequenceId
);
```

**Классификация:** CWE-89 — Improper Neutralization of Special Elements used in an SQL Command.

---

### 8.2. CWE-79: Stored XSS через dangerouslySetInnerHTML

**Где:** `apps/dashboard/src/app/layout.tsx` (строки 82--89 и 92),
`apps/dashboard/src/components/ui/chart.tsx` (строки 89--108),
`packages/template/src/components/elements/ssr-layout-effect.tsx` (строки 13--17).

**Суть:** React экранирует строки в JSX --- это защита. Проблема ---
`dangerouslySetInnerHTML`. В `layout.tsx` вставляются `headTags` из
конфигурации (`NEXT_PUBLIC_STACK_HEAD_TAGS`); в `chart.tsx` --- CSS-переменные;
в `ssr-layout-effect.tsx` скрипт выполняется через `dangerouslySetInnerHTML`.
Если значение `innerHTML` попадает из пользовательских данных без
серверной санитизации --- возможен Stored XSS в браузере администратора.

Реальный код (`layout.tsx:82--92`):

```typescript
// layout.tsx:61 — источник данных из переменной окружения:
const headTags: TagConfigJson[] = JSON.parse(
  getEnvVariable('NEXT_PUBLIC_STACK_HEAD_TAGS', '[]')
);

// layout.tsx:86 — реальный dangerouslySetInnerHTML:
dangerouslySetInnerHTML: { __html: innerHTML ?? "" },

// layout.tsx:92 — inline-скрипт:
<script dangerouslySetInnerHTML={{ __html: "(function(){...})()" }} />
```

**Классификация:** CWE-79 — Improper Neutralization of Input During Web Page Generation.

---

### 8.3. CWE-259 / CWE-798: Захардкоженные секреты в репозитории

**Где:** `apps/backend/.env.development` и `apps/dashboard/.env.development`,
закоммиченные в git. `.github/workflows/lint-and-build.yaml` (строки 36--40)
копирует их в `.env.production.local`.

**Суть:** Оба `.env.development` файла содержат реальные значения
секретов и находятся в репозитории. CI/CD-воркфлоу автоматически копирует
их в production-конфиг. Если CI/CD не переопределяет значения на реальные ---
dev-секреты уйдут в продакшн.

Реальные строки из файлов:

```bash
# apps/dashboard/.env.development:
NEXT_PUBLIC_STACK_PUBLISHABLE_CLIENT_KEY=this-publishable-client-key-is-for-local-development-only
STACK_SECRET_SERVER_KEY=this-secret-server-key-is-for-local-development-only
STACK_FEATUREBASE_JWT_SECRET=secret-value          # строка 15

# apps/backend/.env.development:
STACK_SERVER_SECRET=23-wuNpik0gIW4mruTz25rbIvhuuvZFrLOLtL7J4tyo
CRON_SECRET=mock_cron_secret
STACK_STRIPE_SECRET_KEY=sk_test_mockstripekey
STACK_AWS_ACCESS_KEY_ID=test
STACK_AWS_SECRET_ACCESS_KEY=test
```

CI/CD-воркфлоу (`.github/workflows/lint-and-build.yaml:36--40`):

```yaml
- name: Create .env.production.local file for apps/backend
  run: cp apps/backend/.env.development apps/backend/.env.production.local

- name: Create .env.production.local file for apps/dashboard
  run: cp apps/dashboard/.env.development apps/dashboard/.env.production.local
```

**Классификация:** CWE-259 — Use of Hard-coded Password; CWE-798 — Use of Hard-coded Credentials.

---

### 8.4. CWE-532: Сырой JWT-токен попадает в серверные логи

**Где:** `apps/backend/src/lib/tokens.tsx`, строка 120.

**Суть:** При ошибке разбора JWT в функции `decodeAccessToken` переменная
`accessToken` (полная строка токена) передаётся напрямую в `console.warn`
как объект. Это означает, что в серверных логах (Sentry, stdout, log-агрегатор)
оседают полноценные access-токены пользователей. Токен действителен до
истечения срока (`STACK_ACCESS_TOKEN_EXPIRATION_TIME`), а значит злоумышленник
с доступом к логам может использовать его для аутентификации.

Аналогично строка 117 логирует `decoded?.sub` (userId) и `decoded?.aud`
(projectId) --- структуру токена.

Реальный код (`tokens.tsx:119--121`):

```typescript
} else if (error instanceof JOSEError) {
  console.warn("Unparsable access token. This might be a user error, but if it happens frequently, it's a sign of a misconfiguration.",
    { accessToken, error }  // ← сырой токен в логах
  );
  return Result.error(new KnownErrors.UnparsableAccessToken());
}
```

**Классификация:** CWE-532 — Insertion of Sensitive Information into Log File.

---

### 8.5. CWE-330: Предсказуемый криптографический challenge в Passkey-аутентификации

**Где:** `apps/backend/src/app/api/latest/auth/passkey/initiate-passkey-authentication/route.tsx`, строка 42.
`apps/backend/src/app/api/latest/auth/passkey/initiate-passkey-registration/route.tsx`, строка 48.
`apps/backend/.env.development` — `STACK_ENABLE_HARDCODED_PASSKEY_CHALLENGE_FOR_TESTING=yes`.

**Суть:** Если переменная окружения `STACK_ENABLE_HARDCODED_PASSKEY_CHALLENGE_FOR_TESTING`
установлена, сервер отдаёт фиксированный challenge `"MOCK"` вместо
криптографически случайного значения. Протокол WebAuthn/FIDO2 требует
уникального непредсказуемого challenge на каждую сессию. При известном
challenge злоумышленник может подготовить и воспроизвести валидную
подпись (replay attack), полностью обходя Passkey-аутентификацию.

В `.env.development` (закоммиченном в git) значение этой переменной
равно `yes`, то есть опасный режим включён по умолчанию и может
«просочиться» в staging-среду через CI/CD-воркфлоу.

Реальный код (`initiate-passkey-authentication/route.tsx:39--45`):

```typescript
const authenticationOptions = await generateAuthenticationOptions({
  rpID: "THIS_VALUE_WILL_BE_REPLACED.example.com",
  userVerification: "preferred",
  challenge: getEnvVariable("STACK_ENABLE_HARDCODED_PASSKEY_CHALLENGE_FOR_TESTING", "")
    ? isoUint8Array.fromUTF8String("MOCK")  // ← фиксированный challenge
    : undefined,
  allowCredentials: [],
  timeout: SIGN_IN_TIMEOUT_MS,
});
```

**Классификация:** CWE-330 — Use of Insufficiently Random Values; CWE-294 — Authentication Bypass by Capture-replay.

---

## 9. Оценка рисков

Шкала: вероятность и последствия от 1 до 5. Риск = произведение.
1--4 низкий, 5--9 средний, 10--16 высокий, 17--25 критический.

| Угроза | Вер-ть | Посл-я | Риск | Приоритет | Что делать |
|---|:-:|:-:|:-:|:-:|---|
| Credential stuffing на /password/sign-in | 4 | 4 | 16 (выс.) | 1 | Rate limiting, CAPTCHA после N ошибок, блокировка по IP |
| Dev-секреты в `.env.development` копируются в prod через CI/CD (CWE-259/798) | 4 | 5 | 20 (крит.) | 2 | Убрать `.env.development` из git; secret scanning в CI (Gitleaks); ротация всех попавших в репо ключей |
| Stored XSS через dangerouslySetInnerHTML в dashboard (CWE-79) | 3 | 4 | 12 (выс.) | 3 | Санитизация на сервере, CSP, запрет dangerouslySetInnerHTML для пользовательских данных |
| JWT-токен в серверных логах (CWE-532) | 4 | 4 | 16 (выс.) | 4 | Убрать `accessToken` из аргументов console.warn; логировать только хэш или `[REDACTED]` |
| Фиксированный Passkey challenge → replay attack (CWE-330) | 3 | 5 | 15 (выс.) | 5 | Запретить `STACK_ENABLE_HARDCODED_PASSKEY_CHALLENGE_FOR_TESTING=yes` вне dev; добавить runtime-guard и проверку в CI |
| $queryRawUnsafe без ORM-параметризации (CWE-89) | 2 | 4 | 8 (ср.) | 6 | Заменить на Prisma `$queryRaw` с tagged templates; ужесточить валидацию маппинга |
| Supply chain через npm-зависимости | 2 | 4 | 8 (ср.) | 7 | npm audit, lockfile, Dependabot |
| DoS на тяжёлые эндпоинты | 3 | 3 | 9 (ср.) | 8 | Rate limiting на gateway, лимиты размера запросов |

---

## 10. Вывод

Проанализирован репозиторий stack-auth/stack-auth (ветка main, 31 марта
2026). Построена модель угроз по STRIDE и составлена карта поверхности
атаки.

Главные риски для этого проекта:

**Dev-секреты в продакшне.** `.env.development` закоммичен в репозиторий
и автоматически копируется в `.env.production.local` через CI/CD-воркфлоу
(`.github/workflows/lint-and-build.yaml:36--40`). Риск 20/25 ---
критический. Это самая опасная архитектурная проблема.

**JWT в логах.** `tokens.tsx:120` передаёт сырой access-токен в
`console.warn`. При любом парсинг-сбое (в том числе при атаке перебором)
токен оседает в логах --- в Sentry, stdout и любом подключённом
log-агрегаторе.

**Предсказуемый Passkey challenge.** Переменная
`STACK_ENABLE_HARDCODED_PASSKEY_CHALLENGE_FOR_TESTING=yes` в
закоммиченном `.env.development` активирует статический challenge
`"MOCK"`, делая Passkey-аутентификацию тривиально обходимой через replay
attack.

**Credential stuffing.** Stack Auth --- платформа аутентификации, форма
входа --- цель номер один. Без rate limiting атакующий перебирает пароли
в промышленных масштабах.

**XSS в dashboard.** Платформа мультитенантная --- обслуживает сразу
много приложений. XSS через `dangerouslySetInnerHTML` в layout.tsx может
скомпрометировать сессию администратора.

Приоритетные меры: вынести `.env.development` из git и настроить secret
scanning; убрать токен из логов; добавить runtime-блокировку
небезопасного passkey-режима; внедрить rate limiting на критичных
эндпоинтах; CSP на dashboard.

Prisma ORM и React дают базовую защиту от SQLi и XSS, но только при
правильном использовании. `$queryRawUnsafe` и `dangerouslySetInnerHTML`
обходят эти механизмы, поэтому нужен регулярный аудит кода.

---

*Репозиторий: https://github.com/stack-auth/stack | Ветка: main | Дата анализа: 31.03.2026*
