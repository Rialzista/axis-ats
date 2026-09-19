# Axis ATS — Progress Tracker

Единая точка навигации по всем трём фазам. Ссылки ведут на детальные файлы (там — полный список задач,
файлов/папок к созданию, анти-скоуп). Здесь — только чекбоксы критериев готовности для отметки прогресса.

Обновляется вручную: ставите `[x]` по мере готовности. Порядок фаз — рекомендованный, не обязательный
(например, можно начать бэкенд раньше, если так удобнее — план на это не завязан жёстко).

## Содержание

- [Setup](#setup)
- [Фаза 1 — Frontend (React)](#фаза-1--frontend-react)
  - [День 1 — Scaffold & Theme](#день-1--scaffold--theme)
  - [День 2 — Dashboard](#день-2--dashboard)
  - [День 3 — Candidates](#день-3--candidates)
  - [День 4 — Jobs](#день-4--jobs)
  - [День 5 — Pipeline/Kanban](#день-5--pipelinekanban)
  - [День 6 — Notes/Activity/Reports](#день-6--notesactivityreports)
  - [День 7 — Polish & Deploy](#день-7--polish--deploy)
- [Фаза 2 — Backend (.NET)](#фаза-2--backend-net)
  - [День 8 — Scaffold & Infra](#день-8--scaffold--infra)
  - [День 9 — Domain, Migrations, Seed](#день-9--domain-migrations-seed)
  - [День 10 — Candidates API](#день-10--candidates-api)
  - [День 11 — Jobs & Pipeline API](#день-11--jobs--pipeline-api)
  - [День 12 — Interviews & Reports API](#день-12--interviews--reports-api)
  - [День 13 — Cross-cutting Concerns](#день-13--cross-cutting-concerns)
  - [День 14 — Testing, Deploy, Integration](#день-14--testing-deploy-integration)
- [Фаза 3 — Realtime (SignalR)](#фаза-3--realtime-signalr)
  - [День 15 — SignalR Hub (Backend)](#день-15--signalr-hub-backend)
  - [День 16 — Frontend Realtime Integration](#день-16--frontend-realtime-integration)
- [Фаза 4 — Module Federation (бонус)](#фаза-4--module-federation-бонус)
  - [День 17 — Pipeline Remote](#день-17--pipeline-remote)
- [Справочные файлы](#справочные-файлы)

---

## Setup

Полный список: [`setup.md`](./setup.md)

- [ ] Git установлен, репозиторий инициализирован (`git init` в корне проекта)
- [ ] Node.js LTS (20.x+) установлен
- [ ] WebStorm настроен на проект
- [ ] .NET 8 SDK установлен (`dotnet --version`)
- [ ] Docker Desktop установлен и запущен
- [ ] `dotnet-ef` global tool установлен
- [ ] IDE под C# выбрана и установлена (Rider / VS Code + C# Dev Kit)
- [ ] Postman или `wscat` установлен
- [ ] Аккаунты созданы: GitHub, Vercel/Netlify, Fly.io/Railway/Render

---

## Фаза 1 — Frontend (React)

Обзор: [`00-overview.md`](./00-overview.md) · Модель данных: [`data-model.md`](./data-model.md) ·
Токены: [`design-tokens.md`](./design-tokens.md)

### День 1 — Scaffold & Theme
Детали: [`01-day-1-scaffold-and-theme.md`](./01-day-1-scaffold-and-theme.md)

- [x] `git init` выполнен, `.gitignore` создан
- [ ] Репозиторий создан на GitHub, `origin` подключён, первый пуш сделан
- [ ] `npm run dev` поднимается без ошибок
- [ ] Все 5 пунктов меню кликабельны и меняют URL + контент
- [ ] Активный пункт меню визуально выделен акцентным цветом
- [ ] Цвета/шрифты берутся из CSS-переменных/Tailwind-темы, а не из инлайн-хардкода
- [ ] Нет ни одного `console.error` в браузерной консоли

### День 2 — Dashboard
Детали: [`02-day-2-dashboard.md`](./02-day-2-dashboard.md)

- [ ] Network-таб показывает перехваченные MSW-запросы
- [ ] Dashboard показывает: total candidates, candidates by stage, jobs by status, сравнительный bar chart
- [ ] Изменение мок-данных отражается на дашборде после перезагрузки
- [ ] Loading state виден при первой загрузке
- [ ] Тест на `StatTile` проходит (Vitest+RTL сетап обкатан)

### День 3 — Candidates (список, карточка, форма)
Детали: [`03-day-3-candidates.md`](./03-day-3-candidates.md)

- [ ] Список кандидатов грузится, поиск с debounce работает
- [ ] Фильтры по статусу/источнику комбинируются с поиском (MobX-стор)
- [ ] Сортировка по клику на заголовок колонки
- [ ] Клик по строке открывает `/candidates/:id`
- [ ] Переключение табов в профиле работает без перезагрузки
- [ ] Прямой переход по URL `/candidates/<id>` открывает нужного кандидата
- [ ] `/candidates/new` и `/candidates/:id/edit` работают, форма на RHF+Yup валидирует поля
- [ ] Тесты на `CandidateForm` и `CandidatesFiltersStore` проходят

### День 4 — Jobs (список, карточка, форма)
Детали: [`04-day-4-jobs.md`](./04-day-4-jobs.md)

- [ ] Jobs list грузится, фильтруется, сортируется (MobX-стор)
- [ ] Клик по строке ведёт на `/jobs/:id`
- [ ] Кандидаты на странице вакансии сгруппированы по стадиям
- [ ] Счётчик кандидатов в Jobs list совпадает с реальным числом `PipelineEntry`
- [ ] `/jobs/new` и `/jobs/:id/edit` работают, форма на RHF+Yup валидирует поля
- [ ] Тесты на `JobForm` и `StatusPill` проходят

### День 5 — Pipeline/Kanban
Детали: [`05-day-5-pipeline-kanban.md`](./05-day-5-pipeline-kanban.md)

- [ ] Drag&drop между колонками меняет стадию визуально сразу (optimistic)
- [ ] После F5 изменение сохранилось
- [ ] `KanbanBoard` переиспользуется на `/pipeline` и на `/jobs/:id` без дублирования кода
- [ ] Drop в ту же колонку не ломает состояние
- [ ] Карточка перемещается между колонками с клавиатуры (keyboard sensor)
- [ ] Тест на `useMoveCandidateStage` (optimistic + откат) проходит

### День 6 — Notes/Activity/Reports
Детали: [`06-day-6-notes-activity-reports.md`](./06-day-6-notes-activity-reports.md)

- [ ] Перемещение в Kanban отражается в Changes history без ручного обновления
- [ ] Добавленная заметка появляется в Activity-ленте сразу
- [ ] В Reports можно выбрать поля галочками и увидеть таблицу с этими колонками (MobX-стор)
- [ ] Export CSV скачивает корректный файл
- [ ] Тесты на `exportToCsv` (экранирование запятых) и `ActivityFeedItem` проходят

### День 7 — Polish, Storybook, a11y, Deploy
Детали: [`07-day-7-polish-and-deploy.md`](./07-day-7-polish-and-deploy.md)

- [ ] Публичный URL открывается, все страницы работают без консольных ошибок
- [ ] Прямой переход по глубокой ссылке работает после хард-рефреша
- [ ] Loading/Empty/Error состояния проработаны
- [ ] README даёт полное понимание проекта за 2 минуты
- [ ] Sidebar не разваливает layout на 1280px и уже
- [ ] Storybook поднят, 5–8 компонентов задокументированы историями
- [ ] 2 a11y-теста через `vitest-axe` проходят, фокус визуально виден везде

---

## Фаза 2 — Backend (.NET)

Обзор: [`backend-dotnet/00-overview.md`](./backend-dotnet/00-overview.md) ·
Контракты: [`backend-dotnet/01-domain-and-contracts.md`](./backend-dotnet/01-domain-and-contracts.md)

### День 8 — Scaffold & Infra
Детали: [`backend-dotnet/08-day-8-scaffold-and-infra.md`](./backend-dotnet/08-day-8-scaffold-and-infra.md)

- [ ] `docker-compose up -d` поднимает Postgres (healthy)
- [ ] `dotnet run --project src/Ats.Api` стартует без ошибок
- [ ] `GET /health` возвращает `{ "status": "ok", "db": true }`
- [ ] `/swagger` открывается
- [ ] Решение по enum-сериализации зафиксировано и записано
- [ ] `dotnet user-secrets` настроены, реальный пароль от Postgres нигде не закоммичен

### День 9 — Domain, Migrations, Seed
Детали: [`backend-dotnet/09-day-9-domain-migrations-seed.md`](./backend-dotnet/09-day-9-domain-migrations-seed.md)

- [ ] `dotnet ef database update` проходит на чистой БД
- [ ] После первого запуска БД наполнена сид-данными
- [ ] Повторный запуск не плодит дубликаты
- [ ] Распределение `PipelineEntry` по стадиям неравномерное (проверено SQL-запросом)

### День 10 — Candidates API
Детали: [`backend-dotnet/10-day-10-candidates-api.md`](./backend-dotnet/10-day-10-candidates-api.md)

- [ ] Все эндпоинты Candidates видны и вызываемы через Swagger
- [ ] Фронтенд Candidates list/profile работает против реального бэкенда
- [ ] Заметка через UI видна в Adminer и переживает рестарт фронтенда
- [ ] Поиск/фильтры/сортировка дают тот же результат, что и на MSW

### День 11 — Jobs & Pipeline API
Детали: [`backend-dotnet/11-day-11-jobs-pipeline-api.md`](./backend-dotnet/11-day-11-jobs-pipeline-api.md)

- [ ] Jobs list/detail работают против бэкенда, счётчик кандидатов совпадает
- [ ] Kanban на `/pipeline` и `/jobs/:id` меняет `StageId` в БД
- [ ] После перемещения в БД появляется запись в `activity_events`
- [ ] Перемещение в ту же стадию не создаёт лишних событий

### День 12 — Interviews & Reports API
Детали: [`backend-dotnet/12-day-12-interviews-reports-api.md`](./backend-dotnet/12-day-12-interviews-reports-api.md)

- [ ] Dashboard показывает реальные агрегаты, сверенные вручную через SQL
- [ ] `InterviewsTab` показывает интервью нужного `PipelineEntry`
- [ ] Reports-страница работает без изменений на фронте
- [ ] MSW полностью выключен
- [ ] Все эндпоинты из сводной таблицы реализованы (сверено построчно)

### День 13 — Cross-cutting Concerns + OpenAPI Types
Детали: [`backend-dotnet/13-day-13-cross-cutting-concerns.md`](./backend-dotnet/13-day-13-cross-cutting-concerns.md)

- [ ] Запрос несуществующего id возвращает 404 в формате ProblemDetails
- [ ] `POST .../notes` с пустым текстом возвращает 400 с понятным сообщением
- [ ] Необработанное исключение не роняет процесс, клиенту уходит только `traceId`
- [ ] Swagger UI сгруппирован по тегам с описаниями
- [ ] В логах видна смена стадии кандидата
- [ ] `npm run generate:api-types` генерирует типы из OpenAPI, фронтенд-хуки используют их вместо
  ручных дублирующих интерфейсов

### День 14 — Testing, Deploy, Integration
Детали: [`backend-dotnet/14-day-14-testing-deploy-frontend-integration.md`](./backend-dotnet/14-day-14-testing-deploy-frontend-integration.md)

- [ ] `dotnet test` зелёный, 5–8+ тестов по Candidates/Pipeline
- [ ] Бэкенд задеплоен, `/health` и `/swagger` доступны по прод-URL
- [ ] Прод-фронтенд полностью функционален против прод-бэкенда (весь ручной чеклист пройден)
- [ ] Миграции применены на прод-БД, сид не пересоздаётся при каждом деплое
- [ ] README позволяет поднять весь стек с нуля по инструкции

---

## Фаза 3 — Realtime (SignalR)

Обзор: [`realtime-signalr/00-overview.md`](./realtime-signalr/00-overview.md) ·
Контракт события: [`realtime-signalr/01-event-contract.md`](./realtime-signalr/01-event-contract.md)

### День 15 — SignalR Hub (Backend)
Детали: [`realtime-signalr/15-day-15-signalr-hub-backend.md`](./realtime-signalr/15-day-15-signalr-hub-backend.md)

- [ ] `/hubs/pipeline` принимает WebSocket-подключение
- [ ] `JoinJobGroup`/`LeaveJobGroup` реально меняют состав группы
- [ ] `PATCH /api/pipeline-entries/{id}` триггерит рассылку подключённым клиентам группы
- [ ] `Ats.Application` не зависит напрямую от SignalR-пакета

### День 16 — Frontend Realtime Integration
Детали: [`realtime-signalr/16-day-16-frontend-realtime-integration.md`](./realtime-signalr/16-day-16-frontend-realtime-integration.md)

- [ ] Два окна с разным "Acting as" видят перемещение карточки друг у друга мгновенно
- [ ] Нет мигания/дублирования карточки от эха собственного действия
- [ ] Разрыв и восстановление соединения приводят к автосинхронизации без ручного рефреша
- [ ] Переход между вакансиями корректно покидает старую SignalR-группу

---

## Фаза 4 — Module Federation (бонус)

Строго опционально — только если Дни 1–16 закрыты с запасом. Обзор: [`module-federation-vite/00-overview.md`](./module-federation-vite/00-overview.md)

### День 17 — Pipeline Remote
Детали: [`module-federation-vite/17-day-17-pipeline-remote.md`](./module-federation-vite/17-day-17-pipeline-remote.md)

- [ ] `pipeline-remote` собирается независимо, отдаёт `remoteEntry.js`
- [ ] Хост динамически подгружает Kanban из remote в рантайме
- [ ] React не задублирован в бандлах host/remote
- [ ] Drag&drop и realtime продолжают работать внутри remote
- [ ] Пересборка remote видна в хосте без пересборки хоста (зафиксировано на скриншоте/gif)

---

## Справочные файлы

- [`00-overview.md`](./00-overview.md) — общий обзор Фазы 1, стек, архитектура папок, DoD недели
- [`data-model.md`](./data-model.md) — сущности фронтенда (TS-контракты)
- [`design-tokens.md`](./design-tokens.md) — цвета/шрифты/компоненты из Axis HRM
- [`setup.md`](./setup.md) — установка софта под все три фазы
- [`backend-dotnet/00-overview.md`](./backend-dotnet/00-overview.md) — обзор Фазы 2
- [`backend-dotnet/01-domain-and-contracts.md`](./backend-dotnet/01-domain-and-contracts.md) — сущности бэкенда (C#-контракты)
- [`realtime-signalr/00-overview.md`](./realtime-signalr/00-overview.md) — обзор Фазы 3
- [`realtime-signalr/01-event-contract.md`](./realtime-signalr/01-event-contract.md) — контракт realtime-события
- [`module-federation-vite/00-overview.md`](./module-federation-vite/00-overview.md) — обзор Фазы 4 (бонус)
