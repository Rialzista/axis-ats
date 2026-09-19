# День 16 — Frontend: подключение к SignalR, демо-трюк "Acting as"

## Цель дня

Kanban на фронте живо реагирует на `CandidateStageChanged` от других сессий, без дублирования/мигания
при эхе собственных действий. Добавлен переключатель "Acting as" для убедительного демо без авторизации.

## Пререквизиты

- День 15 завершён (хаб работает, проверен вручную через WS-клиент)

## Файлы и папки к созданию

```
src/
  shared/
    realtime/
      signalrConnection.ts             (синглтон HubConnection + withAutomaticReconnect)
    identity/
      ActingAsStore.ts                 (MobX-класс + localStorage: текущий "actorId"/"actorFullName")
      ActingAsSwitcher.tsx             (UI в Topbar — select из списка users, observer-компонент)
  features/
    pipeline/
      model/
        recentLocalMoves.ts            (короткий TTL-кэш последних локальных action, см. 01-event-contract.md)
      api/
        useJobRealtimeSync.ts          (хук: join/leave группы, обработка входящих событий)
        useMoveCandidateStage.ts       (обновить: передавать actorId из actingAsStore)
  shared/
    ui/
      Sidebar/... (не трогаем)
      Topbar/Topbar.tsx                (обновить: вмонтировать ActingAsSwitcher)
```

## Пошаговые задачи

1. **Пакет**
   - `npm install @microsoft/signalr`

2. **Синглтон-соединение**
   - `shared/realtime/signalrConnection.ts` — один `HubConnection` на всё приложение (не создавать
     новый на каждый компонент): `new HubConnectionBuilder().withUrl(<baseUrl>/hubs/pipeline)
     .withAutomaticReconnect().build()`, экспортировать функцию `getConnection()`, которая лениво
     стартует соединение (`connection.start()`) при первом обращении и переиспользует его дальше
   - Обработать `onreconnecting`/`onreconnected`/`onclose` — хотя бы `console.warn`, чтобы видеть
     разрывы при ручном тестировании (можно временно вырубать Wi-Fi/убивать бэкенд, чтобы проверить
     реальный reconnect, не только в теории)

3. **Acting As**
   - `ActingAsStore.ts` — MobX-класс с персистентностью в `localStorage`: в конструкторе
     `makeAutoObservable(this)` + читать сохранённое значение из `localStorage.getItem(...)`, затем
     `autorun(() => localStorage.setItem(..., JSON.stringify(this.currentUser)))` — MobX `autorun`
     сам переподписывается на используемые observable-поля и синхронизирует их в localStorage при
     каждом изменении, аналог zustand `persist` middleware, но написанный руками (что и есть цель
     недели) — хранит выбранного `User` (id + fullName) из уже существующего списка `useUsers()`
   - `ActingAsSwitcher.tsx` — `Select` в `Topbar`, обёрнутый в `observer()`, при смене — вызывает
     `actingAsStore.setCurrentUser(user)`
   - Дефолт при первом открытии — случайный или первый пользователь из списка, не пустое состояние

4. **useMoveCandidateStage — обновить**
   - Подставлять `actorId` из `ActingAsStore` в тело `PATCH`-запроса (бэкенд уже ждёт его с Дня 15)
   - **Перед вызовом мутации** — записать в `recentLocalMoves` (пункт 5) факт локального действия

5. **Дедупликация эха**
   - `recentLocalMoves.ts` — простая структура (Map или массив) `{ pipelineEntryId, toStageId,
     expiresAt }`, метод `markLocalMove(entryId, stageId)` (TTL ~5 секунд), метод
     `isRecentLocalMove(entryId, stageId): boolean`
   - В обработчике входящего `CandidateStageChanged` (пункт 6): если `isRecentLocalMove(...)` — true,
     обновить только метаданные карточки (кто передвинул, `movedAt`) без полной замены объекта в кэше
     (чтобы React не считал это новым элементом и не дёргал анимацию/скролл)

6. **useJobRealtimeSync**
   - Хук, вызываемый в `PipelinePage`/`JobDetailPage` (там же, где рендерится `KanbanBoard`), принимает
     `jobId`:
     - в `useEffect` при монтировании — `connection.invoke('JoinJobGroup', jobId)`, при размонтировании
       — `connection.invoke('LeaveJobGroup', jobId)` (важно не забыть leave — иначе клиент продолжит
       получать события для вакансий, которые уже не смотрит, и группы на сервере будут расти мусором)
     - подписка `connection.on('CandidateStageChanged', handler)` — **один раз глобально**, не на
       каждый `useJobRealtimeSync`-вызов (если хук используется в нескольких местах одновременно,
       множественные `.on()` на одно и то же событие задвоят обработку) — вынести подписку в
       `signalrConnection.ts` как часть инициализации соединения, а `useJobRealtimeSync` только
       фильтрует события по `jobId` и обновляет `queryClient`
     - обработчик: если `evt.jobId !== jobId` — игнорировать (событие не для этой доски — на случай,
       если соединение получает события от нескольких групп, в которых успело побывать); иначе —
       `queryClient.setQueryData(['job-pipeline-entries', jobId], updater)`, применяя логику
       дедупликации из пункта 5
     - на `connection.onreconnected` — `queryClient.invalidateQueries(['job-pipeline-entries', jobId])`
       (полный рефетч, как описано в `01-event-contract.md`)

7. **Визуальная обратная связь**
   - В `KanbanCard` — показать мелким текстом "moved by {actorFullName}" под `movedAt`, если это поле
     пришло из последнего realtime-события (или всегда, если бэкенд отдаёт это и через обычный
     REST-эндпоинт `GET /api/pipeline-entries` — сверьтесь, что `PipelineEntryDto` из Фазы 2 включает
     информацию об акторе последнего изменения; если нет — это небольшая доработка DTO на бэкенде,
     можно сделать сегодня же)

## Критерии готовности

- [ ] Два окна браузера (разный "Acting as" в каждом) на одной вакансии: перетаскивание в одном
  мгновенно обновляет карточку во втором
- [ ] В окне, где произошло действие, нет визуального мигания/дублирования карточки от эха собственного
  события
- [ ] Разрыв соединения (например, перезапуск бэкенда) и восстановление приводят к автоматическому
  переподключению и досинхронизации доски без ручного обновления страницы пользователем
- [ ] При переходе с одной вакансии на другую (unmount/mount `useJobRealtimeSync`) старая группа
  покидается — проверить логами на бэкенде (День 15, `OnDisconnectedAsync`/явный leave), что группы не
  растут бесконтрольно за сессию

## Не делать сегодня

- Persist истории realtime-событий — не нужно, REST остаётся источником истины
- Распространение того же паттерна на Notes/Activity — отдельная задача, не сегодня (см. anti-scope в
  `01-event-contract.md`)
