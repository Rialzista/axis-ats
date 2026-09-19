# Axis ATS — общий план

## Цель

За 7 дней (реально — 7 вечеров/сессий) собрать рабочий MVP ATS (Applicant Tracking System) на React,
используя визуальный язык концепта [Axis HRM](https://www.behance.net/gallery/225326055/Axis-HRM) (Behance)
как основу дизайна.

Домен — из вашего опыта в CbizSoft/Exelare: вакансии (Jobs), кандидаты (Candidates), воронка найма
(Pipeline/Stages), заметки и активность (Notes/Activity), базовая отчётность (Reports).

Код пишется руками, без генерации компонентов AI. Эти markdown-файлы — только план: что, в каком порядке,
в каких файлах/папках. Там, где нужна опорная спецификация (TS-интерфейсы сущностей, имена пропсов) —
она есть, но это не реализация, а контракт, который вы сами наполняете логикой.

## Технологии

| Область | Выбор | Почему |
|---|---|---|
| Сборка | Vite + React + TypeScript | быстрый старт, HMR, не Angular-CLI — сознательно другой инструмент для тренировки |
| Роутинг | React Router v6 | стандарт де-факто |
| Стили | Tailwind CSS + CSS-переменные (design tokens) | скорость вёрстки без потери контроля над темой |
| Серверное состояние | TanStack Query | кэш, лоадинги/ошибки без ручного boilerplate |
| Клиентское UI-состояние | **MobX** (`mobx` + `mobx-react-lite`) | целевая вакансия использует MobX; на объёме проекта (2–3 стора: фильтры, acting-as, локальный кэш Kanban) разница с Zustand по цене небольшая. Стиль другой: observable-классы + `makeAutoObservable`, не хуки-селекторы — учитывать при написании сторов |
| Формы | React Hook Form + Yup | целевая вакансия — тот же стек; используется в формах создания/редактирования Candidate/Job (Дни 3–4) |
| Таблицы | TanStack Table | сортировка/фильтры/пагинация как в Axis HRM Employees list |
| Графики | Recharts | быстро закрывает Dashboard |
| Drag & Drop | dnd-kit | для Kanban-пайплайна, включая keyboard sensor для a11y |
| Мок-бэкенд | MSW (Mock Service Worker) | перехват fetch/XHR, приближено к реальному API, легко заменить на настоящий бэкенд позже |
| Тесты — **обязательно, по ходу каждого дня** | Vitest + React Testing Library | не откладывается на "если останется время" — 1–3 теста добавляются в каждый день начиная со Дня 2 |
| UI-документация | Storybook | 5–8 компонентов из `shared/ui`, отдельный слот в Дне 7 |
| a11y | ручные практики + `vitest-axe` | семантика, keyboard nav, один автоматический smoke-тест — Дни 5 и 7 |

## Архитектура папок (`src/`)

Feature-based структура (близко к тому, что вы делаете в Angular-модулях, но по-фронтенд-react-конвенции):

```
src/
  app/                      # корень приложения: провайдеры, роутинг, layout-обвязка
    providers/
    routes.tsx
    App.tsx
  pages/                    # route-level страницы, тонкие — собирают features
    DashboardPage/
    CandidatesPage/
    CandidateProfilePage/
    JobsPage/
    JobDetailPage/
    PipelinePage/
    ReportsPage/
  features/                 # бизнес-фичи, каждая самодостаточна
    dashboard/
    candidates/
    jobs/
    pipeline/
    notes-activity/
    reports/
  entities/                 # доменные сущности: типы + мок-данные
    candidate/
    job/
    stage/
    interview/
    note/
    user/
  shared/                   # переиспользуемый слой без знания о домене
    ui/                     # UI-кит (Button, Table, Card, Tabs, Modal, Sidebar, ...)
    lib/                    # хуки, утилиты
    api/                    # http-клиент, MSW-моки
    styles/                 # design tokens, шрифты, tailwind base
    config/                 # константы, роуты-enum
```

Правило зависимостей (как в вашем NgRx-опыте с разделением слоёв): `shared` ничего не знает про `entities`/`features`;
`entities` ничего не знает про `features`; `features` не импортируют друг друга напрямую, только через `pages`.

## Маппинг Axis HRM → ATS

| Axis HRM | ATS-эквивалент | День |
|---|---|---|
| Dashboard | Recruiting Dashboard (воронка, time-to-hire, источники) | 2 |
| Employees list | Candidates list | 3 |
| Employee card (Info/Absences/Teams/Assessment/Documents/Changes history) | Candidate profile (Info/Resume/Activity & Notes/Interviews/Changes history) | 3 |
| — (нет в макете) | Create/Edit Candidate form (RHF + Yup) | 3 |
| — (нет в макете) | Jobs list + Job detail | 4 |
| — (нет в макете) | Create/Edit Job form (RHF + Yup) | 4 |
| — (нет в макете) | Pipeline/Kanban по стадиям | 5 |
| Changes history (в карточке) | Activity timeline (кандидат + вакансия) | 6 |
| Reports-constructor | Reports (упрощённый) | 6 |
| Teams, Mana, Events | — не переносим, вне скоупа ATS | — |

## Definition of Done на конец недели

- [ ] Тёмная тема в стиле Axis HRM применена ко всему приложению (токены, не хардкод цветов по месту)
- [ ] Dashboard с 3–4 виджетами на реальных мок-данных (не статичные картинки)
- [ ] Candidates: список с поиском/фильтрами/сортировкой + карточка кандидата с табами
- [ ] Jobs: список вакансий + карточка вакансии со связанными кандидатами
- [ ] Pipeline: kanban-доска по стадиям, drag&drop меняет стадию кандидата в сторе
- [ ] Notes/Activity: добавление заметки, лента событий по кандидату
- [ ] Reports: минимум один настраиваемый отчёт (фильтр полей + таблица + экспорт в CSV)
- [ ] Create/Edit формы для Candidate и Job на React Hook Form + Yup, с валидацией
- [ ] Vitest-тесты написаны по ходу (не одним блоком в конце) — минимум по 1–2 на Дни 2–6
- [ ] Storybook поднят, 5–8 компонентов `shared/ui` задокументированы историями
- [ ] Базовый a11y: keyboard nav в Kanban, aria-label на icon-only кнопках, 1 автоматический axe-тест
- [ ] Приложение задеплоено (Vercel/Netlify), есть README с описанием и скриншотами

## Как читать дневные файлы

Каждый `NN-day-N-*.md` содержит:
1. **Цель дня** — что должно работать к концу сессии
2. **Пререквизиты** — что должно быть готово с прошлых дней
3. **Файлы и папки к созданию** — точные пути
4. **Пошаговые задачи** — чеклист
5. **Критерии готовности** — как проверить, что день закрыт
6. **Не делать сегодня** — явный анти-скоуп, чтобы не тонуть в деталях раньше времени

Порядок файлов:

- `01-day-1-scaffold-and-theme.md`
- `02-day-2-dashboard.md`
- `03-day-3-candidates.md`
- `04-day-4-jobs.md`
- `05-day-5-pipeline-kanban.md`
- `06-day-6-notes-activity-reports.md`
- `07-day-7-polish-and-deploy.md`
- `data-model.md` — сквозная спецификация сущностей (читать перед днём 2)
- `design-tokens.md` — цвета/шрифты/отступы из Axis HRM (читать перед днём 1)

## Фаза 2 — бэкенд на .NET

После недели фронта (Дни 1–7, MSW-моки) — вторая неделя, заменяющая MSW на настоящий бэкенд на
ASP.NET Core + EF Core + PostgreSQL, повторяющий контракты из `data-model.md`. Фронтенд-код при этом
не меняется, кроме `baseURL`. План: `backend-dotnet/00-overview.md` (Дни 8–14).

## Фаза 3 — Realtime (SignalR)

2 дня поверх готового Pipeline-эндпоинта: live-синхронизация Kanban через SignalR — перемещение
кандидата по стадиям видно во всех открытых сессиях без перезагрузки. План: `realtime-signalr/00-overview.md`
(Дни 15–16).

## Фаза 4 — Module Federation (бонус, опционально)

Делается **только если Дни 1–16 закрыты с запасом времени**. Вынести Pipeline-фичу в отдельный remote
через `@module-federation/vite` (bundler-agnostic Module Federation 2.0, не classic Webpack MF — у вас
уже есть реальный Webpack MF опыт из Angular, повторять его в React не даёт нового сигнала; современный
подход, работающий поверх Vite/Rspack, — то, куда сейчас движется рынок). План: `module-federation-vite/00-overview.md`.
