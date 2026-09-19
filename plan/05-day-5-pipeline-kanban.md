# День 5 — Pipeline: Kanban-доска

## Цель дня

Превратить статичную группировку по стадиям из `JobCandidatesByStage` (День 4) в интерактивный
Kanban с drag&drop, плюс отдельная страница `/pipeline` с выбором вакансии.

## Пререквизиты

- День 4 завершён (`JobCandidatesByStage` показывает кандидатов, сгруппированных по стадиям, без DnD)

## Файлы и папки к созданию

```
src/
  features/
    pipeline/
      api/
        useMoveCandidateStage.ts     (мутация: PATCH pipeline-entry stageId)
      model/
        PipelineBoardStore.ts        (опционально: MobX-класс, локальный optimistic-update стейт колонок)
      ui/
        KanbanBoard/
          KanbanBoard.tsx
          KanbanColumn.tsx
          KanbanCard.tsx
  shared/
    api/
      mockServer/
        handlers/
          pipeline.handlers.ts       (дополнить: PATCH /api/pipeline-entries/:id)
  pages/
    PipelinePage/index.tsx           (выбор вакансии + KanbanBoard)
    JobDetailPage/index.tsx          (обновить: заменить JobCandidatesByStage на KanbanBoard)
```

## Пошаговые задачи

1. **PATCH-эндпойнт**
   - Дополнить `pipeline.handlers.ts`: `PATCH /api/pipeline-entries/:id` принимает `{ stageId }`,
     обновляет запись в мок-массиве (in-memory, живёт до перезагрузки страницы — это нормально для
     проекта, персистентность через localStorage можно добавить в День 7, если останется время)
   - Хендлер должен также обновлять `movedAt` на текущую дату — понадобится, если будете считать
     time-in-stage метрики на дашборде

2. **dnd-kit**
   - Установить `@dnd-kit/core` + `@dnd-kit/sortable` (или обойтись без sortable, если стадии — это
     drop-зоны, а не переупорядочиваемый список внутри колонки — для MVP порядок карточек внутри
     колонки не важен, важна только колонка/стадия)
   - `KanbanBoard.tsx` — оборачивает колонки в `DndContext`, обрабатывает `onDragEnd`: определяет
     новую `stageId` по drop-зоне, вызывает мутацию
   - **a11y**: подключить `KeyboardSensor` из `@dnd-kit/core` в `useSensors` (наравне с
     `PointerSensor`) — dnd-kit из коробки поддерживает перемещение карточек клавиатурой (Tab к
     карточке → Space для захвата → стрелки для перемещения между drop-зонами → Space для отпускания),
     но только если `KeyboardSensor` явно добавлен в сенсоры; без этого шага Kanban доступен только
     мышью, что для тестового a11y-прогона в Дне 7 будет провалом

3. **useMoveCandidateStage**
   - `useMutation` из TanStack Query: на success — инвалидировать `['job-pipeline-entries', jobId]`
     (и опционально `['dashboard-metrics']`, если хотите видеть обновление воронки на дашборде без
     перезагрузки)
   - Реализовать **optimistic update**: карточка визуально переезжает в новую колонку сразу, до ответа
     сервера — это осознанное упражнение на `onMutate`/`onError`/`onSettled` в TanStack Query, полезно
     для тренировки, раз уж цель — делать руками

4. **KanbanColumn / KanbanCard**
   - `KanbanColumn.tsx` — заголовок (имя стадии + счётчик карточек), drop-зона (`useDroppable`)
   - `KanbanCard.tsx` — компактная карточка кандидата: Avatar, имя, currentTitle, дата попадания в
     стадию (`movedAt`) — используется `useDraggable`
   - Визуально терминальные стадии (`isTerminal: true` — Hired/Rejected) можно выделить отдельным цветом
     колонки (зелёный/красный оттенок поверх базовой тёмной темы)

5. **Страница /pipeline**
   - `PipelinePage/index.tsx` — сверху `Select` для выбора вакансии (список из `useJobsList`), ниже —
     `KanbanBoard` для выбранной вакансии. Если вакансия не выбрана — пустое состояние с подсказкой

6. **Обновить Job detail**
   - В `JobDetailPage` заменить `JobCandidatesByStage` на `KanbanBoard` — теперь это не дублирование,
     а один и тот же компонент, используемый в двух местах (важно: `KanbanBoard` не должен знать, что он
     на странице вакансии или на отдельной странице — принимает `jobId` пропом)

7. **Тесты (обязательно)**
   - `useMoveCandidateStage.test.ts` — тест на саму мутацию/optimistic-логику через
     `renderHook` из `@testing-library/react` (не через полный UI): замокать `apiClient`, проверить,
     что при `onMutate` кэш обновляется немедленно, а при ошибке (`onError`) откатывается к прежнему
     состоянию — это ключевая бизнес-логика дня, и она вообще не требует рендера Kanban-доски для теста

## Критерии готовности

- [ ] Перетаскивание карточки кандидата между колонками меняет его стадию визуально сразу (optimistic)
- [ ] После обновления страницы (F5) изменение сохранилось (значит PATCH реально дошёл до мок-хранилища,
  а не только до UI-стейта)
- [ ] Тот же `KanbanBoard` работает и на `/pipeline`, и на `/jobs/:id` без дублирования кода
- [ ] Если перетащить карточку в ту же колонку (no-op) — ничего не ломается, лишний запрос не критичен,
  но желательно не отправлять его вовсе
- [ ] Карточку можно переместить между колонками с клавиатуры (Tab → Space → стрелки → Space), без мыши
- [ ] `npx vitest run` зелёный: тест на `useMoveCandidateStage` (optimistic update + откат при ошибке) проходит

## Не делать сегодня

- Переупорядочивание карточек внутри одной колонки (сортировка по приоритету) — вне скоупа
- Массовое перемещение нескольких кандидатов разом
- Анимации drag&drop сверх того, что dnd-kit даёт из коробки
