# ATS Backend (.NET) — общий план — Фаза 2

## Цель

Продолжение проекта: заменить MSW-моки из Фазы 1 (`plan/01`–`07`) на настоящий бэкенд на
ASP.NET Core, повторяющий контракты, которые уже зафиксированы во фронтенд-плане
(`../data-model.md`, эндпоинты из `shared/api/mockServer/handlers/*`). Фронтенд после этого меняет
только `baseURL` и перестаёт грузить MSW — логика компонентов не трогается.

7 дней (8–14), плотные сессии, как и в Фазе 1. Пишете руками, без генерации кода AI.

## Технологии

| Область | Выбор | Почему |
|---|---|---|
| Платформа | .NET 8 (LTS) | стабильная актуальная версия на момент планирования |
| Web-фреймворк | ASP.NET Core **Minimal API** | современный идиоматичный подход, меньше церемоний чем MVC-контроллеры, хорошо для проекта и для тренировки актуального стиля |
| ORM | EF Core | стандарт де-факто, миграции из коробки |
| БД | PostgreSQL (через Docker) | ближе к реальному проду, чем SQLite; заодно тренировка Docker, который в резюме отмечен как "базовый" — повод углубить |
| Валидация | FluentValidation | явные, тестируемые правила валидации DTO |
| Маппинг Entity↔DTO | **вручную**, extension-методы (`ToDto()`/`ToEntity()`) | сознательно без AutoMapper/Mapster — на проекте ручной маппинг даёт больше контроля и понимания, магия библиотек не нужна для такого объёма сущностей |
| Документация API | Swagger/OpenAPI (`Swashbuckle` или встроенный в .NET 8 `Microsoft.AspNetCore.OpenApi`) | бесплатная документация + фронтенд может сверяться со схемой |
| Тесты | xUnit + `WebApplicationFactory` (integration) | закрывает основные сценарии end-to-end через реальный HTTP pipeline |
| Деплой | Fly.io / Railway / Render (любой с Docker-поддержкой) | бесплатный тир, простой деплой контейнера |

## Архитектура решения

Прагматичная слоистая архитектура — без полного Clean Architecture / CQRS+MediatR оверкилла на
неделю, но с чёткими границами слоёв (то, что вы и так строите в Angular через модули/NgRx):

```
backend/
  Ats.sln
  docker-compose.yml                 # postgres + (опционально) adminer
  src/
    Ats.Api/                         # host: Program.cs, endpoint mapping, middleware, DI
      Endpoints/
      Middleware/
      Program.cs
      appsettings.json
      appsettings.Development.json
    Ats.Application/                 # use-case сервисы, DTO, интерфейсы, валидаторы — без знания про EF
      Candidates/
      Jobs/
      Pipeline/
      Interviews/
      Notes/
      Activity/
      Reports/
      Common/
    Ats.Domain/                      # чистые сущности и enum'ы, без зависимостей от фреймворков
      Entities/
      Enums/
    Ats.Infrastructure/              # EF Core: DbContext, конфигурации, миграции, репозитории, сидинг
      Persistence/
        AtsDbContext.cs
        Configurations/
        Migrations/
      Seed/
  tests/
    Ats.Api.Tests/
```

Правило зависимостей: `Domain` ничего не знает ни о ком. `Application` знает только `Domain` (через
интерфейсы репозиториев, реализация — в `Infrastructure`). `Api` знает `Application` и `Infrastructure`
(только в `Program.cs` для DI-регистрации). `Infrastructure` знает `Domain` и `Application` (реализует
его интерфейсы). Это зеркалит принцип, который вы применяете с NgRx: бизнес-логика не должна знать
про транспорт и про конкретную БД.

## Маппинг фронтенд-контракта → бэкенд

Сущности 1:1 берутся из `../data-model.md` (Candidate, Job, Stage, PipelineEntry, Interview, Note,
ActivityEvent, User) — они становятся EF-сущностями в `Ats.Domain/Entities`, а TS-интерфейсы на
фронте остаются DTO-контрактом, которому бэкенд обязан соответствовать по форме JSON.

Эндпоинты, которые уже спроектированы под MSW и теперь нужно реализовать по-настоящему:

| Метод | Путь | Фронтенд-потребитель |
|---|---|---|
| GET | `/api/candidates` (query: search, status, source, sort, page) | `useCandidatesList` |
| GET | `/api/candidates/{id}` | `useCandidate` |
| GET | `/api/candidates/{id}/notes` | `useCandidateActivity` |
| POST | `/api/candidates/{id}/notes` | `useAddNote` |
| GET | `/api/candidates/{id}/activity` | `useCandidateActivity` |
| GET | `/api/jobs` (query: search, department, status, sort, page) | `useJobsList` |
| GET | `/api/jobs/{id}` | `useJob` |
| GET | `/api/pipeline-entries?jobId=` | `useJobPipelineEntries` |
| PATCH | `/api/pipeline-entries/{id}` (body: `{ stageId }`) | `useMoveCandidateStage` |
| GET | `/api/dashboard/metrics` | `useDashboardMetrics` |
| GET | `/api/stages` | (используется в фильтрах/Kanban для списка колонок) |
| GET | `/api/interviews?pipelineEntryId=` | `InterviewsTab` |
| GET | `/api/users` | (для hiring manager / author в заметках) |

Если в процессе Фазы 1 контракт где-то отличался от этой таблицы — ориентир **фронтенд-код**, эта
таблица только справочная, сверяйте по реальным MSW-хендлерам.

## Definition of Done Фазы 2

- [ ] `docker-compose up` поднимает Postgres, приложение подключается к нему по строке подключения из
  `appsettings.Development.json`
- [ ] Все эндпоинты из таблицы выше реализованы и возвращают JSON той же формы, что ожидает фронтенд
- [ ] Миграции EF Core применяются командой, БД содержит сид-данные сопоставимого объёма с фронтенд-моками
- [ ] Фронтенд из Фазы 1 работает против реального бэкенда без единого изменения в UI-компонентах
  (меняется только базовый URL и убирается инициализация MSW)
- [ ] Swagger доступен по `/swagger`, все эндпоинты в нём задокументированы
- [ ] Минимум 5–8 интеграционных тестов покрывают основные сценарии (список с фильтрами, смена стадии,
  добавление заметки)
- [ ] Бэкенд задеплоен на публичный URL, фронтенд на проде смотрит на него

## Порядок файлов

- `01-domain-and-contracts.md` — сквозная спецификация EF-сущностей и DTO (аналог `data-model.md` из
  Фазы 1, читать перед Днём 8)
- `08-day-8-scaffold-and-infra.md`
- `09-day-9-domain-migrations-seed.md`
- `10-day-10-candidates-api.md`
- `11-day-11-jobs-pipeline-api.md`
- `12-day-12-interviews-reports-api.md`
- `13-day-13-cross-cutting-concerns.md`
- `14-day-14-testing-deploy-frontend-integration.md`
