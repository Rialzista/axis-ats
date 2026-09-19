# День 15 — SignalR Hub на бэкенде

## Цель дня

`PipelineHub` подключён, клиенты могут вступать/выходить из групп по вакансии, сервер рассылает
`CandidateStageChanged` при успешной смене стадии. Проверка через `wscat`/Postman WebSocket-клиент
или временный тестовый HTML, без готового фронтенда (он завтра).

## Пререквизиты

- Фаза 2 полностью завершена (Дни 8–14), `MoveStageAsync` работает и создаёт `ActivityEvent`
- Прочитаны `00-overview.md` и `01-event-contract.md` этой фазы

## Файлы и папки к созданию

```
src/
  Ats.Api/
    Hubs/
      PipelineHub.cs
    Program.cs                           (обновить: AddSignalR, MapHub, CORS для credentials)
  Ats.Application/
    Pipeline/
      Dto/
        CandidateStageChangedEvent.cs
      IPipelineRealtimeNotifier.cs        (абстракция над IHubContext — см. пункт 3)
  Ats.Infrastructure/
    Realtime/
      SignalRPipelineNotifier.cs          (реализация IPipelineRealtimeNotifier поверх IHubContext<PipelineHub>)
```

## Пошаговые задачи

1. **Пакет и регистрация**
   - SignalR входит в `Microsoft.AspNetCore.App` shared framework — отдельный NuGet-пакет для сервера
     не нужен, только `builder.Services.AddSignalR()` в `Program.cs`
   - `app.MapHub<PipelineHub>("/hubs/pipeline")`

2. **PipelineHub**
   - `Hubs/PipelineHub.cs`, наследник `Hub` (не `Hub<T>` — для этой задачи строготипизированный клиент
     не обязателен, но можно использовать `Hub<IPipelineClient>` с интерфейсом, объявляющим
     `Task CandidateStageChanged(CandidateStageChangedEvent evt)`, если хочется компилируемой проверки
     сигнатур вместо `Clients.Group(...).SendAsync("CandidateStageChanged", evt)` со строкой — решите,
     что важнее: строгая типизация (`Hub<T>`) или простота (обычный `Hub` + магическая строка)
   - Методы:
     ```csharp
     public async Task JoinJobGroup(string jobId)
         => await Groups.AddToGroupAsync(Context.ConnectionId, $"job-{jobId}");

     public async Task LeaveJobGroup(string jobId)
         => await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"job-{jobId}");
     ```
   - Логировать `OnConnectedAsync`/`OnDisconnectedAsync` (override, вызвать `base`) — полезно видеть в
     логах, сколько живых соединений, особенно при отладке reconnect-сценариев завтра

3. **Абстракция уведомлений — зачем**
   - `Ats.Application` не должен зависеть от `Microsoft.AspNetCore.SignalR` напрямую (это
     web-специфичный пакет, а `Application` в вашей слоистой архитектуре — транспорт-агностичный слой,
     тот же принцип, что и с репозиториями в Фазе 2) — поэтому объявляем интерфейс
     `IPipelineRealtimeNotifier.NotifyStageChangedAsync(CandidateStageChangedEvent evt)` в `Application`,
     а реализацию `SignalRPipelineNotifier` (использующую `IHubContext<PipelineHub>`) — в `Infrastructure`
     (единственном месте, знающем про конкретный транспорт)
   - Зарегистрировать в DI: `services.AddScoped<IPipelineRealtimeNotifier, SignalRPipelineNotifier>()`

4. **Событие**
   - `CandidateStageChangedEvent` — поля точно по контракту из `01-event-contract.md`
     (`pipelineEntryId, jobId, candidateId, candidateFullName, fromStageId, toStageId, movedAt, actorId,
     actorFullName`)
   - В `PipelineService.MoveStageAsync` (Фаза 2, День 11) — после успешного `SaveChangesAsync`, вызвать
     `await _realtimeNotifier.NotifyStageChangedAsync(evt)` с уже собранными данными (кандидат, актор
     уже загружены для формирования `ActivityEvent` — переиспользовать, не делать лишний запрос к БД)

5. **actorId в запросе**
   - `MoveStageRequest` (Фаза 2) пока не содержит `actorId` — если ещё не добавляли для
     `ActivityEvent.ActorId`, добавить сейчас: `{ stageId: Guid, actorId: Guid }`. Если на бэкенде уже
     был захардкожен какой-то системный актор — заменить на реальный из запроса (понадобится для
     демо-трюка "Acting as" завтра на фронте)

6. **CORS для SignalR**
   - SignalR с WebSocket-транспортом требует `AllowCredentials()` в CORS-политике, что несовместимо с
     `AllowAnyOrigin()` — origin должен быть явно перечислен (уже должно быть так с Фазы 2, Дня 8, но
     проверить и при необходимости поправить: `policy.WithOrigins("http://localhost:5173")
     .AllowAnyHeader().AllowAnyMethod().AllowCredentials()`)

7. **Ручная проверка без фронтенда**
   - Использовать `wscat -c ws://localhost:5000/hubs/pipeline` или Postman (поддерживает SignalR
     с недавних версий) — вручную дёрнуть `PATCH /api/pipeline-entries/{id}` через Swagger параллельно с
     открытым WS-соединением, убедиться, что сообщение долетает до подключённого клиента группы

## Критерии готовности

- [ ] `/hubs/pipeline` принимает WebSocket-подключение
- [ ] `JoinJobGroup`/`LeaveJobGroup` реально добавляют/убирают из SignalR-группы (проверить логами —
  залогировать количество вызовов или ConnectionId)
- [ ] `PATCH /api/pipeline-entries/{id}` триггерит рассылку `CandidateStageChanged` подключённым к
  соответствующей группе клиентам — подтверждено вручную через WS-клиент
- [ ] `Ats.Application` не имеет прямой зависимости от SignalR-пакета (проверить `.csproj` /
  `using`-директивы — зависимость только через собственный интерфейс)

## Не делать сегодня

- Никакого фронтенд-кода — только бэкенд и ручная проверка через WS-клиент/Postman
- Presence, множественные хабы — вне скоупа этой фазы (см. `01-event-contract.md`)
