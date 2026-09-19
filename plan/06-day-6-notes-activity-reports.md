# День 6 — Notes/Activity лента и Reports

## Цель дня

Наполнить заглушки `ActivityTab`/`ChangesHistoryTab` из Дня 3 реальной лентой событий + заметками,
и собрать упрощённый Reports-конструктор (аналог Reports из Axis HRM).

## Пререквизиты

- День 5 завершён (перемещение по стадиям генерирует события — сегодня эти события должны попасть
  в ленту активности)

## Файлы и папки к созданию

```
src/
  features/
    notes-activity/
      api/
        useCandidateActivity.ts      (объединённый запрос: Note + ActivityEvent, отсортировано по дате)
        useAddNote.ts                (mutation)
      ui/
        ActivityFeed/
          ActivityFeed.tsx
          ActivityFeedItem.tsx       (рендер в зависимости от type: note / stage_changed / interview_...)
        AddNoteForm/AddNoteForm.tsx
    reports/
      model/
        ReportBuilderStore.ts        (MobX-класс: выбранная сущность, выбранные колонки, фильтры)
      ui/
        ReportFieldsPicker/
          ReportFieldsPicker.tsx     (чекбоксы: какие поля включить в отчёт)
        ReportTable/
          ReportTable.tsx            (переиспользует shared/ui/DataTable)
        ExportCsvButton/
          ExportCsvButton.tsx
      lib/
        exportToCsv.ts               (чистая функция: data + columns -> CSV string -> download)
        exportToCsv.test.ts
  shared/
    api/
      mockServer/
        handlers/
          notes.handlers.ts          (GET/POST /api/candidates/:id/notes)
          activity.handlers.ts       (GET /api/candidates/:id/activity)
  pages/
    ReportsPage/index.tsx
    CandidateProfilePage/            (обновить ActivityTab/ChangesHistoryTab, подключить реальные данные)
```

## Пошаговые задачи

1. **Дописать событийность на предыдущих днях**
   - В `useMoveCandidateStage` (День 5) при успешном PATCH — также создавать `ActivityEvent` с
     `type: 'stage_changed'` и `payload: { from, to }` через MSW handler (добавить в тот же хендлер
     логику записи события в мок-массив activity). Это ключевой момент: лента активности должна быть
     живой, а не захардкоженной один раз при генерации моков

2. **Notes API**
   - `notes.handlers.ts`: `GET /api/candidates/:id/notes`, `POST /api/candidates/:id/notes`
     (добавляет в мок-массив, возвращает созданную запись)
   - `useAddNote.ts` — mutation, на success инвалидирует `['candidate-activity', candidateId]`

3. **ActivityFeed**
   - `useCandidateActivity.ts` — объединяет `Note[]` и `ActivityEvent[]` в один массив, сортирует по
     `createdAt` desc (можно сделать на клиенте после двух отдельных запросов, или одним агрегированным
     MSW-эндпойнтом — решите сами, оба варианта valid)
   - `ActivityFeedItem.tsx` — switch по типу события: для `note` — текст + автор, для `stage_changed` —
     "Moved from {from} to {to}", для `interview_scheduled` — дата/тип интервью и т.д. (как Changes
     history в макете: дата, тип, кто изменил)
   - `AddNoteForm.tsx` — textarea + кнопка Submit, вызывает `useAddNote`

4. **Подключить в Candidate profile**
   - `ActivityTab.tsx` теперь рендерит `AddNoteForm` + `ActivityFeed` (все события)
   - `ChangesHistoryTab.tsx` рендерит `ActivityFeed`, но отфильтрованный только на системные события
     (без `type: 'note_added'`) — переиспользуем один компонент с параметром фильтрации, не копируем код

5. **Reports — модель**
   - `ReportBuilderStore.ts` — MobX-класс: `selectedEntity: 'candidates' | 'jobs'`,
     `selectedFields: string[]`, `filters: Record<string, unknown>`, action-методы (`setEntity`,
     `toggleField`, `setFilter`) — тот же паттерн `makeAutoObservable`, что и в Днях 3–5
   - `ReportFieldsPicker.tsx` — список доступных полей для выбранной сущности (захардкодить два набора:
     поля Candidate и поля Job) с чекбоксами

6. **Reports — таблица и экспорт**
   - `ReportTable.tsx` — берёт данные через уже существующие `useCandidatesList`/`useJobsList` (не
     плодить новый эндпойнт), проецирует только выбранные `selectedFields` в колонки `DataTable`
   - `exportToCsv.ts` — чистая функция без UI-зависимостей: принимает массив объектов + список полей,
     возвращает CSV-строку; отдельно — утилита скачивания через `Blob`/`URL.createObjectURL`
   - `ExportCsvButton.tsx` — вызывает `exportToCsv` на текущих отфильтрованных данных

7. **Страница Reports**
   - `pages/ReportsPage/index.tsx` — переключатель сущности (Candidates/Jobs), `ReportFieldsPicker`,
     `ReportTable`, `ExportCsvButton`

8. **Тесты (обязательно)**
   - `exportToCsv.test.ts` — чистая функция, идеальный кандидат для unit-теста без моков и без DOM:
     проверить экранирование запятых/кавычек внутри значений (частый источник багов в CSV-экспорте —
     значение с запятой должно быть в кавычках), корректный порядок колонок по переданному списку полей
   - `ActivityFeedItem.test.tsx` — рендер с событием `type: 'stage_changed'` показывает текст
     "Moved from X to Y", рендер с `type: 'note_added'` показывает текст заметки

## Критерии готовности

- [ ] Перемещение кандидата в Kanban (День 5) тут же отражается новой записью в его Changes history
  без ручного обновления страницы
- [ ] Можно добавить заметку кандидату, она появляется в Activity-ленте сразу после сабмита
- [ ] В Reports можно переключить сущность, выбрать 3–4 поля галочками, увидеть таблицу именно с этими
  колонками
- [ ] Кнопка Export CSV скачивает файл, который открывается в Excel/Numbers с корректными колонками
- [ ] `npx vitest run` зелёный: тесты на `exportToCsv` (включая экранирование запятых) и
  `ActivityFeedItem` проходят

## Не делать сегодня

- Сохранение конфигурации отчёта как шаблон (в макете есть "Save as template" — красиво, но не критично)
- Экспорт в PDF/XLSX — только CSV, этого достаточно для демонстрации идеи
- Real-time обновления через WebSocket — polling/инвалидация через TanStack Query достаточно
