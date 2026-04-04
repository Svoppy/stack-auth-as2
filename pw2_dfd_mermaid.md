# DFD for Stack Auth

Основа анализа: в качестве исследуемой системы берется `apps/backend` проекта Stack Auth, так как именно там реализованы маршруты аутентификации, OAuth, email-очередь, webhooks, session replay, работа с PostgreSQL, S3 и ClickHouse.

Подтверждающие участки кода:
- `stack-auth/apps/backend/package.json`
- `stack-auth/apps/backend/src/app/api/latest/auth/password/sign-in/route.tsx`
- `stack-auth/apps/backend/src/app/api/latest/auth/oauth/authorize/[provider_id]/route.tsx`
- `stack-auth/apps/backend/src/app/api/latest/auth/oauth/callback/[provider_id]/route.tsx`
- `stack-auth/apps/backend/src/lib/emails.tsx`
- `stack-auth/apps/backend/src/lib/webhooks.tsx`
- `stack-auth/apps/backend/src/s3.tsx`
- `stack-auth/apps/backend/src/lib/events.tsx`

Этот файл можно вставить в Mermaid Live Editor или в diagrams.net:
- Mermaid Live Editor: https://mermaid.live/
- diagrams.net: Insert -> Advanced -> Mermaid

## Контекстная DFD

```mermaid
flowchart LR
  subgraph TB1["TB1 Public / Untrusted Zone"]
    E1["E1 Конечный пользователь"]
    E2["E2 Dashboard / клиентское приложение"]
  end

  subgraph SYS["Stack Auth Backend"]
    P0["P0 Система аутентификации и управления пользователями"]
  end

  subgraph TB3["TB3 Third-party Services"]
    E3["E3 OAuth-провайдер"]
    E4["E4 Email-провайдер"]
    E5["E5 Webhook-потребитель"]
  end

  E1 -->|F1 HTTPS REST<br/>логин, регистрация, OTP, passkey| P0
  P0 -->|F2 HTTPS response<br/>token, cookie, session data| E1

  E2 -->|F3 HTTPS API<br/>users, teams, projects, RBAC| P0
  P0 -->|F4 HTTPS response<br/>CRUD result, config, profiles| E2

  P0 -->|F5 HTTPS redirect / backchannel<br/>OAuth authorize| E3
  E3 -->|F6 HTTPS callback / token exchange<br/>code, tokens, user info| P0

  P0 -->|F7 SMTP or HTTPS API<br/>verification, reset, invite email| E4
  P0 -->|F8 HTTPS POST / Svix<br/>signed webhook events| E5
```

## DFD Уровня 1

```mermaid
flowchart TB
  subgraph TB1["TB1 Public / Untrusted Zone"]
    E1["E1 Конечный пользователь"]
    E2["E2 Dashboard / клиентское приложение"]
  end

  subgraph SYS["Stack Auth Backend"]
    direction TB

    subgraph PROC["Процессы"]
      direction LR
      P1["P1 Аутентификация и сессии"]
      P2["P2 OAuth broker"]
      P3["P3 Users / Teams / RBAC"]
      P4["P4 Email / verification / queue"]
      P5["P5 Events / Webhooks / Session Replay"]
    end

    subgraph TB2["TB2 Data Zone"]
      direction LR
      D1[("D1 PostgreSQL / Prisma")]
      D2[("D2 S3 private storage")]
      D3[("D3 ClickHouse events")]
    end
  end

  subgraph TB3["TB3 Third-party Services"]
    E3["E3 OAuth-провайдер"]
    E4["E4 Email-провайдер"]
    E5["E5 Webhook-потребитель"]
  end

  E1 -->|F1 HTTPS REST<br/>email, password, OTP, passkey| P1
  P1 -->|F2 HTTPS response<br/>access token, refresh token, cookie| E1
  P1 <-->|F3 SQL / Prisma<br/>users, auth methods, sessions, refresh tokens| D1

  E1 -->|F4 HTTPS GET / redirect<br/>start OAuth login/link| P2
  P2 -->|F5 HTTPS redirect<br/>client_id, state, PKCE| E3
  E3 -->|F6 HTTPS callback + token exchange<br/>code, tokens, user info| P2
  P2 <-->|F7 SQL / Prisma<br/>oauth state, linked accounts, oauth tokens| D1

  E2 -->|F8 HTTPS API<br/>users, teams, projects, permissions| P3
  P3 -->|F9 HTTPS response<br/>CRUD result, config, profiles| E2
  P3 <-->|F10 SQL / Prisma<br/>users, teams, projects, permissions| D1

  P1 -->|F11 internal trigger<br/>verify/reset/invite| P4
  P3 -->|F12 internal trigger<br/>admin emails / invitations| P4
  P4 <-->|F13 SQL / Prisma<br/>email outbox, templates, verification codes| D1
  P4 -->|F14 SMTP/TLS or HTTPS API<br/>email payload| E4

  E1 -->|F15 HTTPS JSON<br/>session replay batches| P5
  P1 -->|F16 internal events<br/>auth/session activity| P5
  P3 -->|F17 internal events<br/>user/team/project changes| P5
  P5 <-->|F18 SQL / Prisma<br/>replay metadata, webhook config| D1
  P5 -->|F19 S3 API over HTTPS<br/>replay chunks, media objects| D2
  P5 -->|F20 ClickHouse client<br/>audit, security, usage events| D3
  P5 -->|F21 HTTPS POST / Svix<br/>signed webhook events| E5
```

## Идентификаторы

- Внешние сущности: `E1`, `E2`, `E3`, `E4`, `E5`
- Процессы: `P0` для контекстной диаграммы; `P1`-`P5` для уровня 1
- Хранилища: `D1`, `D2`, `D3`
- Границы доверия: `TB1`, `TB2`, `TB3`
- Потоки данных: `F1`-`F21`

## Что делать дальше

- Если преподаватель требует строгую контекстную DFD, на контекстной диаграмме оставить только `P0` и внешние сущности.
- Если преподаватель требует единый номер процесса, можно заменить `P0` на `P1`, а на уровне 1 использовать `P1.1`-`P1.5`.
- Для отчета рядом с диаграммой стоит добавить таблицу потоков `F1-F21` с типом данных, направлением, протоколом и уровнем доверия источника.
