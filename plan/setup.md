# Setup — что установить перед стартом

macOS (Darwin), команды через Homebrew где уместно. Пройтись сверху вниз перед Днём 1.

## Обязательно для всех фаз

| Инструмент | Версия | Установка | Зачем |
|---|---|---|---|
| Git | любая свежая | `brew install git` (скорее всего уже стоит) | контроль версий, деплой через GitHub |
| Node.js | LTS 20.x+ | `brew install node` или через `nvm` (лучше `nvm`, если планируете переключать версии между проектами) | Vite, npm-пакеты, весь фронт |
| npm | идёт с Node | — | пакетный менеджер (можно pnpm/yarn по желанию, план написан под npm-команды) |
| WebStorm | текущая | уже установлен | основная IDE для фронта |

Проверка: `node -v`, `npm -v`, `git --version`.

## Фаза 1 — Frontend (React)

Ничего сверх обязательного списка — все зависимости (`react`, `vite`, `tailwindcss`,
`@tanstack/react-query`, `@tanstack/react-table`, `mobx`, `mobx-react-lite`, `react-hook-form`, `yup`,
`recharts`, `@dnd-kit/core`, `msw`, `@faker-js/faker`, `storybook`, `vitest-axe` и т.д.) ставятся через
`npm install` внутри проекта, отдельного системного софта не требуют.

Опционально:
- `lucide-react` — иконки (упомянуто в Дне 1 как вариант)
- Расширение браузера **React Developer Tools** (Chrome Web Store) — сильно ускоряет отладку
  компонентов и TanStack Query DevTools (`@tanstack/react-query-devtools`, npm-пакет, не браузерное
  расширение)

## Фаза 2 — Backend (.NET)

| Инструмент | Версия | Установка | Зачем |
|---|---|---|---|
| .NET SDK | 8.0 LTS | `brew install --cask dotnet-sdk` или установщик с dotnet.microsoft.com | компиляция и запуск ASP.NET Core |
| Docker Desktop | текущая | `brew install --cask docker`, затем запустить приложение вручную минимум раз (принять лицензию, дать доступ) | PostgreSQL + Adminer в контейнерах |
| dotnet-ef (global tool) | под версию SDK | `dotnet tool install --global dotnet-ef` | миграции EF Core (`dotnet ef migrations add`, `dotnet ef database update`) |
| IDE для C# | — | **JetBrains Rider** (рекомендую, раз уже привычны к WebStorm — тот же вендор, похожий UX, отличная поддержка EF Core/Docker) либо VS Code + расширение **C# Dev Kit** (бесплатно) | разработка бэкенда |

Проверка: `dotnet --version` (должен показать 8.x), `docker --version`, `docker compose version`,
`dotnet ef --version` (после установки tool).

Опционально:
- **Adminer** — не устанавливается отдельно, поднимается как контейнер из `docker-compose.yml`
  (Дню 8), альтернатива — **TablePlus** (`brew install --cask tableplus`, есть бесплатный лимит) или
  **DBeaver** (`brew install --cask dbeaver-community`), если хочется постоянный GUI-клиент к Postgres
  вне контейнера
- **Postman** (`brew install --cask postman`) — удобнее голого `curl`/Swagger UI для ручного тестирования
  эндпоинтов, пригодится и для Фазы 3 (Postman умеет WebSocket/SignalR-запросы)

## Фаза 3 — Realtime (SignalR)

Сверх Фазы 1+2 ничего нового по установке — `@microsoft/signalr` ставится через `npm install` во
фронтенд-проекте, SignalR на бэкенде входит в ASP.NET Core shared framework (уже есть с .NET SDK).

Для ручной проверки хаба без фронта (День 15):
- **Postman** (см. выше) — поддерживает SignalR/WebSocket-подключения в современных версиях, либо
- `wscat` — `npm install -g wscat` (низкоуровневый, но лёгкий вариант, если не хочется ставить Postman)

## Для Дня 7 / Дня 14 (деплой)

Аккаунты (не софт, но нужны заранее — лучше завести не в последний день):
- **GitHub** — репозиторий, из него разворачивают и фронт, и бэк
- **Vercel** или **Netlify** — хостинг фронтенда (бесплатный тир достаточен)
- **Fly.io**, **Railway** или **Render** — хостинг Docker-контейнера с .NET API + managed Postgres
  (бесплатный тир может требовать привязку карты у некоторых провайдеров — уточнить на месте, если
  критично, выбрать тот, что не требует)

CLI по выбранному хостингу (опционально, можно и через веб-интерфейс):
- Vercel: `npm install -g vercel`
- Fly.io: `brew install flyctl`
- Railway: `brew install railway`

## Итоговый порядок установки (по шагам)

1. `brew install git node` (или Node через nvm)
2. Установить/проверить WebStorm
3. `brew install --cask dotnet-sdk` (или официальный installer)
4. `brew install --cask docker` → запустить Docker Desktop вручную, дождаться "Docker is running"
5. `dotnet tool install --global dotnet-ef`
6. Установить Rider (`brew install --cask rider`) или VS Code + C# Dev Kit
7. `brew install --cask postman` (или `npm install -g wscat`)
8. Завести аккаунты: GitHub, Vercel/Netlify, Fly.io/Railway/Render — до Дня 7/14, не обязательно сейчас

После этого — можно начинать `01-day-1-scaffold-and-theme.md`.
