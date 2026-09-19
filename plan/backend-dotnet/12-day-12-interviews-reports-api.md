# День 12 — Interviews, Dashboard metrics, Reports API

## Цель дня

Оставшиеся эндпоинты из сводной таблицы (`00-overview.md`): Interviews (read-only), Dashboard metrics
(агрегаты), Users, и поддержка Reports с бэкенда (тот же список Candidates/Jobs, но с проекцией полей).
Фронтенд Дней 2 и 6 (Фаза 1) полностью переезжает на реальный бэкенд — Фаза 1 тем самым закрыта целиком.

## Пререквизиты

- День 11 завершён

## Файлы и папки к созданию

```
src/
  Ats.Application/
    Interviews/
      Dto/InterviewDto.cs
      IInterviewsService.cs
      InterviewsService.cs
    Dashboard/
      Dto/DashboardMetricsDto.cs
      IDashboardService.cs
      DashboardService.cs
    Users/
      Dto/UserDto.cs
    Reports/
      ReportQuery.cs                       (entity: "candidates" | "jobs", fields: string[])
  Ats.Api/
    Endpoints/
      InterviewsEndpoints.cs
      DashboardEndpoints.cs
      UsersEndpoints.cs
      ReportsEndpoints.cs
```

## Пошаговые задачи

1. **Interviews**
   - `InterviewDto`: id, pipelineEntryId, scheduledAt, interviewerIds, type, scorecard (вложенный
     объект, может быть `null`)
   - `InterviewsService.GetByPipelineEntryIdAsync(Guid pipelineEntryId)` → `List<InterviewDto>`
   - `GET /api/interviews?pipelineEntryId=` — read-only, создание интервью вне скоупа (как и во фронтенд
     Дне 3 Фазы 1, `InterviewsTab` был только на чтение)

2. **Users**
   - `UserDto`: id, fullName, avatarUrl, role
   - `GET /api/users` — простой список, без фильтров, нужен фронту для отображения имён вместо голых
     `hiringManagerId`/`authorId` там, где ещё не заджойнено на бэкенде

3. **Dashboard metrics — агрегаты**
   - `DashboardMetricsDto`: минимум под виджеты из Фазы 1 Дня 2:
     - `totalCandidates: int`, `newCandidatesLastPeriod: int` (условно — за последние 30 дней, дата
       отсечки — на ваше усмотрение, зафиксировать явно в коде комментарием почему именно так)
     - `candidatesByStage: List<{ stageId, stageName, count }>` — группировка `PipelineEntry` по
       `StageId` (это и есть "воронка")
     - `jobsByStatus: List<{ status, count }>`
     - `candidatesBySource: List<{ source, count }>`
   - `DashboardService.GetMetricsAsync()` — несколько отдельных LINQ-запросов с `GroupBy`/`CountAsync`,
     не пытаться выразить всё одним гигантским запросом — читаемость важнее (для объёма данных
     проекта разница в производительности незаметна)
   - `GET /api/dashboard/metrics`

4. **Reports — проекция полей**
   - Reports на фронте (Фаза 1 День 6) уже переиспользовал `useCandidatesList`/`useJobsList` — то есть
     отдельный бэкенд-эндпоинт для Reports может не понадобиться вовсе, если фронт просто продолжает
     запрашивать полные списки и сам проецирует нужные поля на клиенте (`selectedFields`)
   - Если объём данных вырастет настолько, что гонять на клиент лишние поля станет ощутимо (для
     проекта — вряд ли) — можно добавить `GET /api/reports?entity=candidates&fields=fullName,email`
     с server-side проекцией через `.Select()` с динамическим построением выражения. **Не делайте это
     заранее** — только если реально понадобится; в противном случае просто зафиксируйте в этом файле
     решение "Reports переиспользует существующие /api/candidates и /api/jobs без изменений"

5. **Подключение фронтенда (финал Фазы 1 → Фаза 2 интеграции)**
   - Фаза 1 День 2: `useDashboardMetrics` → `/api/dashboard/metrics`
   - Фаза 1 День 6: `useCandidateActivity`/`useAddNote` уже подключены с Дня 10; убедиться, что
     `ActivityFeed` корректно показывает события `StageChanged` от Дня 11 вперемешку с `NoteAdded`
   - Полностью убрать инициализацию MSW из `main.tsx` (Фаза 1, День 2) — она больше не нужна ни для
     одного эндпоинта

## Критерии готовности

- [ ] Dashboard на фронте показывает реальные агрегаты из Postgres, цифры совпадают с ручным подсчётом
  через SQL в Adminer (проверить хотя бы `totalCandidates` и одну строку `candidatesByStage`)
- [ ] `InterviewsTab` кандидата показывает интервью, связанные именно с его `PipelineEntry`
- [ ] Reports-страница на фронте продолжает работать без изменений (если решение — переиспользовать
  существующие списочные эндпоинты)
- [ ] MSW полностью выключен, в Network-таб браузера видно реальные HTTP-запросы к `localhost:5000`
- [ ] Все эндпоинты из таблицы в `00-overview.md` реализованы — сверить чеклист построчно

## Не делать сегодня

- Создание/редактирование Interview — только чтение, как и договаривались в Фазе 1
- Кастомный билдер отчётов с server-side агрегацией сверх простой проекции — не оправдано объёмом данных
