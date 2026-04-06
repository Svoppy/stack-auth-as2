**ПРАКТИЧЕСКАЯ РАБОТА №1**

**Моделирование угроз и составление карты поверхности атаки**

**Вариант 1**

**CSE-2506\**

Выполнил: Казиханов Диас

Проверил: Лисневский Р. В.

**1. Исходные данные**

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
(STACK_SECRET_SERVER_KEY), OAuth-конфиги (client_id/client_secret),
данные сессий, профили, API-ключи проектов.

# **2. Область анализа**

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

**Переменные окружения** --- NEXT_PUBLIC_STACK_PROJECT_ID,
NEXT_PUBLIC_STACK_PUBLISHABLE_CLIENT_KEY, STACK_SECRET_SERVER_KEY.

# **3. Критичные активы и нарушители**

## **3.1. Активы**

Учётные записи --- email, хеши паролей, OAuth-привязки. Потеря ---
полный захват аккаунта.

JWT-токены --- access (короткоживущий) и refresh (долгоживущий). Утечка
refresh-токена даёт длительный доступ без пароля.

Серверные секреты --- STACK_SECRET_SERVER_KEY, ключ подписи JWT, OAuth
client_secret. Если утекут --- атакующий получит полный контроль над
аутентификацией.

Cookies сессий --- управляются Stack Auth, привязаны к пользователю.

Профильные данные --- имя, email, аватар, метаданные организаций.

Конфигурация проектов --- API-ключи (publishable/secret), настройки
OAuth, разрешённые домены.

## **3.2. Нарушители**

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

# **4. Доверительные границы**

В архитектуре Stack Auth четыре ключевых границы доверия:

**Граница 1: Браузер --- Frontend.** Браузер --- недоверенная среда.
Всё, что приходит от клиента (формы, параметры), считается потенциально
вредоносным. Клиентская валидация --- только для UX.

**Граница 2: Frontend --- Backend API.** Связь через HTTPS. Backend
проверяет JWT, CORS, CSRF. Publishable key виден всем (NEXT_PUBLIC\_\*),
а secret key --- только на сервере.

**Граница 3: Backend --- PostgreSQL.** Prisma ORM параметризует запросы.
БД в отдельном Docker-контейнере, доступ только с сервера.

**Граница 4: Backend --- OAuth-провайдеры.** Backend обменивается с
Google, GitHub и т.д. через client_id/client_secret. Ответы
верифицируются, но зависимость от внешних сервисов --- отдельный вектор.

Доверенная зона: серверные процессы, БД, переменные окружения.
Недоверенная: браузер, сеть, внешние провайдеры, npm-зависимости.

# **5. Модель угроз STRIDE**

  -----------------------------------------------------------------------------------------------------
  **Компонент**         **Категория**   **Угроза**       **Источник**   **Последствия**   **Риск**
  --------------------- --------------- ---------------- -------------- ----------------- -------------
  Форма входа           Spoofing        Перебор паролей  Внешний        Захват чужих      Высокий
  (/password/sign-in)                   (credential                     аккаунтов         
                                        stuffing) через                                   
                                        автоматические                                    
                                        запросы                                           

  JWT access-токен      Tampering       Подделка payload Внешний        Эскалация         Высокий
                                        при слабой                      привилегий        
                                        подписи или                                       
                                        утечке ключа                                      

  Профиль (/profile)    Info Disclosure IDOR ---         Внешний        Утечка email и    Средний
                                        получение данных                имён              
                                        чужого профиля                                    
                                        через перебор ID                                  

  send-reset-code /     Repudiation     Нет логов        Внешний        Невозможность     Средний
  reset                                 запросов сброса                 расследования     
                                        --- нельзя                                        
                                        восстановить                                      
                                        картину                                           
                                        инцидента                                         

  Серверные логи        Info Disclosure Токены или       Внутренний     Извлечение        Высокий
                                        пароли попадают                 секретов из логов 
                                        в логи открытым                                   
                                        текстом                                           

  PostgreSQL            DoS             Нет rate         Внешний        Отказ в           Средний
                                        limiting ---                    обслуживании      
                                        массовые запросы                                  
                                        кладут базу                                       

  Dashboard             Elevation of    Прямые           Внешний        Контроль чужих    Критический
                        Privilege       API-вызовы в                    проектов          
                                        обход интерфейса                                  
                                        дают                                              
                                        админ-доступ                                      

  OAuth callback        Spoofing        Подмена          Внешний        Вход под чужим    Высокий
                                        redirect_uri для                аккаунтом         
                                        перехвата                                         
                                        authorization                                     
                                        code                                              
  -----------------------------------------------------------------------------------------------------

# **6. Карта поверхности атаки**

  --------------------------------------------------------------------------------------------------------------------------------------------
  **Точка входа**                             **Тип         **Компонент**     **Доверенность**   **Куда идут    **Защита**     **Угрозы**
                                              данных**                                           данные**                      
  ------------------------------------------- ------------- ----------------- ------------------ -------------- -------------- ---------------
  /api/latest/auth/password/sign-in           email, пароль Backend API       Нет                Prisma -\>     HTTPS,         Credential
                                                                                                 PostgreSQL     валидация      stuffing, SQLi

  /api/latest/auth/password/sign-up           email,        Backend API       Нет                Prisma -\>     Валидация      Массовая
                                              пароль, имя                                        PostgreSQL     схемы          регистрация,
                                                                                                                               слабые пароли

  /api/latest/auth/password/send-reset-code   email         Backend API       Нет                Токен -\> SMTP Rate limiting  Перебор email,
                                                                                                                               enumeration

  /api/latest/users/me                        JSON (имя,    Backend API       Частично (JWT)     Prisma\        JWT-проверка   XSS, IDOR
                                              аватар)                                            БД, HTML                      

  Cookies (refresh)                           Токен         Браузер/Backend   Нет                HTTP headers   HttpOnly,      Кража через
                                                                                                                Secure,        XSS, CSRF
                                                                                                                SameSite       

  JWT (Authorization)                         Access token  Backend API       Частично           Верификация    HMAC/RSA       Replay, утечка
                                                                                                 подписи                       ключа

  URL (redirect_uri, code)                    Строки        OAuth handler     Нет                Обмен code на  Whitelist URI  Open redirect,
                                                                                                 token                         CSRF

  .env / env vars                             Секреты       Сервер            Да                 Внутреннее     .gitignore     Утечка в CI/CD

  Webhook endpoints                           JSON          Backend API       Внешний сервис     Обработка      HMAC-подпись   Подделка
                                                                                                 событий                       запросов

  package.json / lockfile                     Зависимости   Сборка            npm registry       node_modules   lockfile,      Supply chain,
                                                                                                                audit          typosquatting
  --------------------------------------------------------------------------------------------------------------------------------------------

# **7. Анализ потоков данных**

## **7.1. Логин и пароль -\> SQL-запрос**

**Source:** пользователь вводит email и пароль в форму (React-компонент
SignIn), формируется JSON.

**Propagation: POST /api/latest/auth/password/sign-in
(apps/backend/src/app/api/latest/auth/password/sign-in/route.tsx).
Backend разбирает тело через yup-схему (emailSchema, passwordSchema).**

**Sanitization:** Prisma параметризует запросы автоматически.
Дополнительно --- zod-валидация. Но если кто-то напишет raw query через
\$queryRawUnsafe со строковой конкатенацией --- защита не сработает.

**Sink: comparePassword (строка 54) сравнивает пароль с хешем из БД с
защитой от timing-атак. При успехе --- createAuthTokens (строки 71-74)
создаёт refresh + access JWT-токены (tokens.tsx:374-382).**

**Фрагмент реального кода (sign-in/route.tsx):**

// sign-in/route.tsx:53-55 --- timing-safe сравнение:

if (!await comparePassword(password, passwordAuthMethod?.passwordHash
\|\| \"\")) {

throw new KnownErrors.EmailPasswordMismatch();

}

// sign-in/route.tsx:71-74 --- генерация токенов:

const { refreshToken, accessToken } = await createAuthTokens({

tenancy, projectUserId: contactChannel.projectUser.projectUserId,

});

## **7.2. Поле профиля -\> HTML (XSS)**

**Source:** пользователь заполняет display name через форму
редактирования профиля.

**Propagation: PATCH /api/latest/users/me
(apps/backend/src/app/api/latest/users/me/route.tsx). Данные сохраняются
в БД, потом отображаются в dashboard и компонентах SDK.**

**Sanitization: React экранирует строки в JSX. Но в
apps/dashboard/src/app/layout.tsx (строки 82-89 и 92) и
apps/dashboard/src/components/ui/chart.tsx (строки 89-108) реально
используется dangerouslySetInnerHTML --- места риска.**

**Sink:** DOM браузера другого пользователя или админа. Stored XSS ---
скрипт работает в чужой сессии.

**Фрагмент реального кода (layout.tsx:82-89):**

{headTags.map((tag, index) =\> {

const { tagName, attributes, innerHTML } = tag;

return React.createElement(tagName, {

dangerouslySetInnerHTML: { \_\_html: innerHTML ?? \"\" },

\...attributes,

});

})}; // layout.tsx:82-89

## **7.3. Email для сброса пароля -\> токен и письмо**

**Source:** email в форме сброса пароля.

**Propagation: POST /api/latest/auth/password/send-reset-code ---
отправляет письмо; POST /api/latest/auth/password/reset --- применяет
код. Одинаковый ответ независимо от наличия пользователя.**

**Sanitization: email валидируется через yup emailSchema. Защита от
enumeration: одинаковый ответ {\"success\": \"maybe, only if user with
e-mail exists\"} в обоих ветках (send-reset-code/route.tsx строки 50-57
и 77-82).**

**Sink: verification code сохраняется в БД, письмо уходит через
sendEmailFromDefaultTemplate (verification-code-handler.tsx:38-49). В
dev-режиме --- Inbucket (apps/backend/.env.development:86-87). Пароль
обновляется через adminUpdate (строки 61-67
verification-code-handler.tsx).**

**Фрагмент реального кода (send-reset-code/route.tsx:50-57):**

// send-reset-code/route.tsx:50-57 --- anti-enumeration:

if (!contactChannel) {

await wait(2000 + Math.random() \* 1000);

return { statusCode: 200, body: { success: \"maybe, only if user with
e-mail exists\" } };

}

// Строки 77-82: тот же ответ при успехе --- email enumeration
невозможен

# **8. Выявленные уязвимости**

## **8.1. CWE-89: SQL Injection**

**Де: apps/backend/src/lib/external-db-sync.ts (строки 526-565) ---
функция syncExternalDb использует \$queryRawUnsafe с параметрами
tenancyId и lastSequenceId.**

**Суть: Prisma параметризует обычные запросы, но \$queryRawUnsafe ---
нет. Строка fetchQuery формируется внешним маппингом. Частичная защита
--- проверка плейсхолдеров \$1 и \$2 (строки 531-535). Некорректное
использование плейсхолдеров этой проверкой не выявляется.**

Реальный код (external-db-sync.ts:564):

// external-db-sync.ts:564 --- реальный вызов \$queryRawUnsafe:

const rows = await
internalPrisma.\$replica().\$queryRawUnsafe\<any\[\]\>(

fetchQuery, tenancyId, lastSequenceId

);

// Проверка плейсхолдеров \$1/\$2 перед вызовом (строки 531-535):

## **8.2. CWE-79: XSS**

**Де: apps/dashboard/src/app/layout.tsx (строки 82-89 и 92),
apps/dashboard/src/components/ui/chart.tsx (строки 89-108),
packages/template/src/components/elements/ssr-layout-effect.tsx (строки
13-17).**

**Суть: React экранирует строки в JSX. Проблема ---
dangerouslySetInnerHTML. В layout.tsx вставляются headTags из
конфигурации; в chart.tsx --- CSS-переменные; в ssr-layout-effect.tsx
скрипт выполняется через eval и dangerouslySetInnerHTML. Конкретный XSS
не доказан, но эти места обходят стандартную защиту React.**

Реальный код (layout.tsx:82-89, 92):

// layout.tsx:86 --- реальный dangerouslySetInnerHTML:

dangerouslySetInnerHTML: { \_\_html: innerHTML ?? \"\" },

// layout.tsx:92 --- inline-скрипт темизации:

\<script dangerouslySetInnerHTML={{ \_\_html: \"(function(){\...})()\"
}} /\>

## **8.3. CWE-259: Захардкоженные секреты**

**Де: apps/dashboard/.env.development (строки 6-8, 15),
apps/backend/.env.development (строки 81-88),
docker/dependencies/docker.compose.yaml (строки 8-10, 161-163).**

**Суть: в репо лежат .env.development с dev-ключами. В
.github/workflows/lint-and-build.yaml (строки 36-40) dev-файлы
копируются в .env.production.local. Если CI/CD не переопределяет
значения --- dev-секреты попадут в продакшн.**

Реальный код (apps/dashboard/.env.development):

\# apps/dashboard/.env.development (строки 6-8, 15):

NEXT_PUBLIC_STACK_PUBLISHABLE_CLIENT_KEY=this-publishable-client-key-is-for-local-development-only

STACK_SECRET_SERVER_KEY=this-secret-server-key-is-for-local-development-only

## **8.4. CWE-352: CSRF**

**Де:
apps/backend/src/app/api/latest/auth/oauth/callback/\[provider_id\]/route.tsx
(строки 100-106) --- CSRF-защита реализована для browser-redirect
OAuth-флоу через cookie.**

**Суть: OAuth-флоу защищён CSRF-cookie stack-oauth-inner-{innerState}:
устанавливается при authorize (строки 178-187), проверяется и удаляется
при callback (строки 100-106). JSON-mode использует PKCE. Эндпоинты вне
OAuth-флоу (смена пароля, профиль) требуют отдельной проверки
CSRF-защиты.**

Реальная CSRF-защита (oauth/callback/\[provider_id\]/route.tsx:100-106):

// oauth/callback/\[provider_id\]/route.tsx:100-106 --- проверка
CSRF-cookie:

if (outerInfo.responseMode !== \"json\") {

const cookieInfo = (await cookies()).get(\"stack-oauth-inner-\" +
innerState);

if (cookieInfo?.value !== \"true\") { throw new StatusError(400, \"CSRF
cookie missing\"); }

## **8.5. CWE-862: Отсутствие авторизации**

**Де: apps/backend/src/app/api/latest/project-permissions/crud.tsx
(строки 23-96),
apps/backend/src/app/api/latest/team-permissions/crud.tsx (строки
25-103).**

**Суть: Stack Auth поддерживает роли и permission graph. Клиент может
видеть только свои права (строки 74-79 / 80-86). Многотенантная граница
через model Tenancy (schema.prisma:60-84) и model ProjectUser
(schema.prisma:247-280).**

Реальный код (project-permissions/crud.tsx:74-79):

// project-permissions/crud.tsx:74-79 --- проверка прав клиента:

async onList({ auth, query }) {

if (auth.type === \"client\") {

const currentUserId = auth.user?.id \|\| throwErr(new
KnownErrors.CannotGetOwnUserWithoutUser());

if (query.user_id !== currentUserId) {

throw new StatusError(StatusError.Forbidden, \"Client can only list
permissions for their own user\");

}

# **9. Оценка рисков**

Шкала: вероятность и последствия от 1 до 5. Риск = произведение. 1-4
низкий, 5-9 средний, 10-16 высокий, 17-25 критический.

  -------------------------------------------------------------------------------------------------------
  **Угроза**          **Вер-ть**   **Посл-я**   **Риск**   **Приоритет**   **Что делать**
  ------------------- ------------ ------------ ---------- --------------- ------------------------------
  Credential stuffing 4            4            16 (выс.)  1               Rate limiting, CAPTCHA после N
  на                                                                       ошибок, блокировка по IP
  /password/sign-in                                                        

  Утечка JWT signing  2            5            10 (выс.)  2               Ротация ключей, хранение в
  key                                                                      vault, мониторинг

  Stored XSS через    3            4            12 (выс.)  3               Санитизация на сервере, CSP,
  профиль                                                                  запрет dangerouslySetInnerHTML

  CSRF на мутирующих  3            3            9 (ср.)    4               SameSite=Strict, CSRF-токены,
  эндпоинтах                                                               проверка Origin

  IDOR на API         3            4            12 (выс.)  5               Проверка принадлежности
  ресурсов                                                                 ресурса пользователю на каждом
                                                                           эндпоинте

  Dev-секреты в проде 2            5            10 (выс.)  6               CI/CD-проверка: не деплоить с
                                                                           dev-ключами

  Supply chain через  2            4            8 (ср.)    7               npm audit, lockfile,
  npm                                                                      Dependabot

  DoS на тяжёлые      3            3            9 (ср.)    8               Rate limiting на gateway,
  эндпоинты                                                                лимиты размера запросов
  -------------------------------------------------------------------------------------------------------

# **10. Вывод**

Проанализирован репозиторий stack-auth/stack-auth (ветка main, 18 марта
2026). Построена модель угроз по STRIDE и составлена карта поверхности
атаки.

Главные риски для этого проекта:

Credential stuffing. Stack Auth --- платформа аутентификации, форма
входа --- цель номер один. Без rate limiting атакующий перебирает пароли
в промышленных масштабах.

Утечка серверных секретов. Если ключ подписи JWT утечёт, атакующий
сможет выпускать любые токены. В репо уже лежат dev-ключи --- их
случайное использование в проде будет фатальным.

XSS и IDOR. Платформа мультитенантная --- обслуживает сразу много
приложений. XSS в dashboard может скомпрометировать админскую сессию.
IDOR даст доступ к данным чужого проекта.

Приоритетные меры: rate limiting на критичных эндпоинтах, ротация
секретов при деплое, проверка авторизации на каждом API-маршруте,
серверная санитизация ввода, CSP, аудит зависимостей.

Prisma ORM и React дают базовую защиту от SQLi и XSS, но только при
правильном использовании. Raw queries и dangerouslySetInnerHTML обходят
эти механизмы, поэтому нужен регулярный аудит кода.
