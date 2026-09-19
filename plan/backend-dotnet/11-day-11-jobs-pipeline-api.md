# День 11 — Jobs API + Pipeline API (включая смену стадии)

## Цель дня

Jobs и Pipeline полностью на реальном бэкенде — включая PATCH смены стадии кандидата, который должен
порождать `ActivityEvent` (то же требование, что было во фронтенд-плане на День 5/6 Фазы 1, теперь
источник истины — сервер). Фронтенд Дней 4–5 (Фаза 1) переключается на реальные эндпоинты.

## Пререквизиты

- День 10 завершён (Candidates API работает, паттерн Service+Endpoints обкатан)

## Файлы и папки к созданию

```
src/
  Ats.Application/
    Jobs/
      Dto/
        JobListItemDto.cs
        JobDetailsDto.cs
      JobsQuery.cs
      IJobsService.cs
      JobsService.cs
      JobMappingExtensions.cs
    Pipeline/
      Dto/
        PipelineEntryDto.cs
        MoveStageRequest.cs
      IPipelineService.cs
      PipelineService.cs
      PipelineMappingExtensions.cs
  Ats.Api/
    Endpoints/
      JobsEndpoints.cs
      PipelineEndpoints.cs
      StagesEndpoints.cs
```

## Пошаговые задачи

1. **Jobs — DTO и сервис**
   - `JobListItemDto`: id, title, department, location, employmentType, status, hiringManagerName,
     openedAt, candidatesCount (посчитанное через `PipelineEntries.Count` — агрегат прямо в запросе
     через `.Select(...)`, не отдельным запросом на каждую вакансию — N+1 здесь классическая ошибка,
     явно её избежать через один `GroupJoin`/подзапрос)
   - `JobDetailsDto`: все поля + `HiringManager` как вложенный мини-DTO (`{ id, fullName, avatarUrl }`)
   - `JobsService.GetListAsync(JobsQuery)` — тот же паттерн фильтрации/сортировки, что и Candidates
     (search по Title, фильтр по Department/Status)
   - `JobsService.GetByIdAsync(Guid id)`

2. **Stages — простой read-only эндпоинт**
   - `StagesEndpoints.cs`: `GET /api/stages` — просто отдать все `Stage` отсортированные по `Order`,
     без отдельного Service-слоя ради одного простого запроса (обратиться к `AtsDbContext` прямо в
     endpoint-делегате — не всё обязано проходить через Service, если логики нет вообще)

3. **Pipeline — DTO и сервис**
   - `PipelineEntryDto`: id, candidateId, candidate (мини: fullName, avatarUrl, currentTitle), jobId,
     stageId, movedAt, rejectionReason — именно этот DTO рендерится в карточке Kanban на фронте, поэтому
     сразу включает данные кандидата, чтобы фронт не делал второй запрос на каждую карточку
   - `PipelineService.GetByJobIdAsync(Guid jobId)` → `List<PipelineEntryDto>`
   - `PipelineService.MoveStageAsync(Guid entryId, MoveStageRequest request)`:
     1. Найти `PipelineEntry`, если нет — 404
     2. Проверить, что `request.StageId` существует в `Stages` — если нет, 400 (не доверять входным
        данным вслепую)
     3. Запомнить старое `StageId` (для события)
     4. Обновить `StageId` и `MovedAt = DateTime.UtcNow`
     5. Создать `ActivityEvent` с `Type = StageChanged`, `PayloadJson = JsonSerializer.Serialize(new {
        from = oldStageName, to = newStageName })`, `CandidateId = entry.CandidateId`
     6. Сохранить всё **в одной транзакции** (`SaveChangesAsync` после обоих изменений — EF Core сам
        обернёт в транзакцию, если оба `Add`/`Update` в одном `SaveChanges`, но если логика разъезжается
        на два вызова `SaveChangesAsync` — оборачивайте явно через `context.Database
        .BeginTransactionAsync()`, чтобы не потерять консистентность при сбое между шагами)

4. **Endpoints**
   - `JobsEndpoints.cs`: `GET /api/jobs`, `GET /api/jobs/{id:guid}`
   - `PipelineEndpoints.cs`: `GET /api/pipeline-entries?jobId=`, `PATCH /api/pipeline-entries/{id:guid}`
   - Зарегистрировать все три группы (`Jobs`, `Pipeline`, `Stages`) в `Program.cs`

5. **Подключение фронтенда**
   - Фаза 1 Дни 4–5: переключить `useJobsList`, `useJob`, `useJobPipelineEntries`,
     `useMoveCandidateStage` на реальные эндпоинты
   - Особое внимание на **optimistic update** из Дня 5 Фазы 1: теперь есть реальная задержка сети,
     а не мгновенный MSW-ответ — хороший момент проверить, что optimistic UI действительно скрывает
     задержку, а не просто визуально "работал и так" на нулевой MSW-латентности. Можно временно добавить
     искусственную задержку в `MoveStageAsync` (`Task.Delay(500)`) на время ручного тестирования, потом
     убрать

## Критерии готовности

- [ ] Jobs list и Job detail на фронте работают против бэкенда, счётчик кандидатов в списке вакансий
  совпадает с реальным количеством `PipelineEntry`
- [ ] Kanban на `/pipeline` и на `/jobs/:id` тянет данные с бэкенда и после drag&drop реально меняет
  `StageId` в БД (проверить в Adminer)
- [ ] После перемещения карточки в БД появилась новая запись в `activity_events` с корректным payload
- [ ] Повторное перемещение в ту же стадию не создаёт дублирующихся бессмысленных событий (проверить
  логику: если `oldStageId == newStageId`, можно вообще не писать событие и не обновлять `MovedAt`)

## Не делать сегодня

- Interviews, Reports — завтра
- Валидацию через FluentValidation — базовых `if`-проверок пока достаточно, полноценная валидация День 13
