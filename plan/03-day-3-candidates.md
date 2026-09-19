# День 3 — Candidates: список, карточка, создание/редактирование

## Цель дня

Полноценный `CandidatesPage` со списком, поиском, фильтрами, сортировкой (аналог Employees list в
Axis HRM), `CandidateProfilePage` с табами (аналог Employee card), и форма создания/редактирования
кандидата на React Hook Form + Yup.

> Дня в этом файле фактически на полтора-два вечера — добавлена форма создания/редактирования (её не
> было в исходном скоупе, добавлена под требования конкретной вакансии, см. `PROGRESS.md`). Если не
> укладываетесь за один вечер — это нормально, разделите на список+профиль (пункты 1–6) и форму
> (пункт 7) двумя сессиями, не режьте по живому середину пункта.

## Пререквизиты

- День 2 завершён (данные, API, TanStack Query подключены)
- Установлены `mobx`, `mobx-react-lite` (если ещё не ставили — это первый день, где появляется стор)

## Файлы и папки к созданию

```
src/
  shared/
    ui/
      DataTable/
        DataTable.tsx               (обёртка над @tanstack/react-table, максимально generic)
        DataTable.types.ts
      SearchInput/SearchInput.tsx
      Badge/Badge.tsx
      Avatar/Avatar.tsx
      Tabs/Tabs.tsx
      Select/Select.tsx             (для фильтров: статус, источник)
  features/
    candidates/
      api/
        useCandidatesList.ts        (query с параметрами: search, status, source, sort, page)
        useCandidate.ts             (query по id)
        useCreateCandidate.ts       (mutation)
        useUpdateCandidate.ts       (mutation)
      model/
        CandidatesFiltersStore.ts   (MobX-класс: search, statusFilter, sourceFilter, selectedIds)
        candidateFormSchema.ts      (Yup-схема валидации)
      ui/
        CandidatesTable/
          CandidatesTable.tsx
          columns.tsx               (определение колонок TanStack Table)
        CandidatesFilters/
          CandidatesFilters.tsx
        CandidateCard/
          CandidateCard.tsx         (шапка профиля: аватар, имя, статус, теги)
        CandidateTabs/
          InfoTab.tsx
          ResumeTab.tsx
          ActivityTab.tsx           (заглушка — наполнится в День 6)
          InterviewsTab.tsx
          ChangesHistoryTab.tsx
        CandidateForm/
          CandidateForm.tsx         (RHF-форма, общая для create и edit)
          CandidateForm.test.tsx
  pages/
    CandidatesPage/index.tsx        (обновить)
    CandidateProfilePage/index.tsx  (обновить)
    CandidateCreatePage/index.tsx
    CandidateEditPage/index.tsx
```

## Пошаговые задачи

1. **DataTable (generic)**
   - Установить `@tanstack/react-table`
   - `shared/ui/DataTable/DataTable.tsx` принимает `columns`, `data`, опционально `onRowClick`,
     `enableRowSelection` — не завязан на Candidate, переиспользуется для Jobs (День 4) и Reports (День 6)
   - Поддержать: сортировку по клику на заголовок (стрелка `↕`, как в макете), чекбоксы для
     множественного выбора строк

2. **Candidates list — данные**
   - `features/candidates/model/CandidatesFiltersStore.ts` — MobX-класс с `makeAutoObservable(this)` в
     конструкторе: поля `search: string`, `statusFilter: CandidateStatus | null`,
     `sourceFilter: CandidateSource | null`, `selectedIds: string[]`, action-методы
     (`setSearch`, `setStatusFilter`, `toggleSelected`, `reset`) — экземпляр создаётся один раз (модуль
     или React Context) и потребляется через `observer()` из `mobx-react-lite` в компонентах, которые
     на него подписаны (это отличие от Zustand-хуков: подписка идёт через оборачивание компонента в
     `observer`, а не через селектор внутри хука)
   - `features/candidates/api/useCandidatesList.ts` — `useQuery`, ключ включает текущие значения из
     стора (`['candidates', store.search, store.statusFilter, store.sourceFilter]`) — MobX сам не
     триггерит перерендер хука вне `observer`, поэтому компонент, вызывающий `useCandidatesList`,
     должен быть обёрнут в `observer`, иначе смена фильтра не вызовет новый запрос
   - Запрос идёт в MSW handler с этими query-параметрами (handler уже подготовлен в День 2 — сегодня
     реализовать в нём реальную фильтрацию/сортировку по массиву)

3. **Candidates list — UI**
   - `CandidatesFilters.tsx` — `SearchInput` (с `useDebouncedValue` из Дня 2) + два `Select` (статус,
     источник) + кнопка Reset — как блок фильтров в макете ("Status: Active x", "Job: Product Designer x")
   - `columns.tsx` — колонки: чекбокс, Avatar+Full name, Email, Current title, Status (Badge), Source,
     Tags (несколько Badge), дата создания
   - `CandidatesTable.tsx` собирает `DataTable` + клик по строке ведёт на `/candidates/:id`
   - `pages/CandidatesPage/index.tsx` — заголовок страницы, `CandidatesFilters`, `CandidatesTable`,
     обработка `isLoading`/пустого состояния ("No candidates match your filters")

4. **Candidate profile — шапка**
   - `CandidateCard.tsx` — левая колонка карточки (как в макете): большой Avatar, имя, теги
     (currentTitle + source как Badge), базовые поля (email, phone, location) — статичный layout,
     данные из `useCandidate(id)`

5. **Candidate profile — табы**
   - `shared/ui/Tabs/Tabs.tsx` — generic компонент табов (принимает список `{ id, label }` и
     `activeTabId`/`onChange`), без завязки на конкретные табы
   - Табы: Info, Resume, Activity, Interviews, Changes history — как в Employee card из макета
     (Info/Absences/Teams/Assessment/Documents/Changes history), но под ATS-домен
   - `InfoTab.tsx` — развёрнутые поля кандидата (всё, что не влезло в шапку)
   - `ResumeTab.tsx` — на сегодня просто плейсхолдер "Resume preview coming soon" + `resumeUrl` как
     ссылка, если есть — полноценный парсинг резюме вне скоупа недели
   - `ActivityTab.tsx`, `ChangesHistoryTab.tsx` — заглушки с текстом "Will be implemented on Day 6" —
     явно фиксируем, что это осознанно отложено, а не забыто
   - `InterviewsTab.tsx` — список `Interview` кандидата (просто таблица/список, без создания — только
     чтение, т.к. `Interview` сущность и моки уже есть с Дня 2)

6. **Роутинг деталей**
   - `pages/CandidateProfilePage/index.tsx` читает `:id` из `useParams`, дергает `useCandidate(id)`,
     собирает `CandidateCard` + `Tabs` + активный контент таба

7. **Create/Edit Candidate — форма (RHF + Yup)**
   - Установить `react-hook-form`, `yup`, `@hookform/resolvers`
   - `features/candidates/model/candidateFormSchema.ts` — Yup-схема: `fullName` required,
     `email` required + формат email, `phone` опционален но с форматной проверкой если заполнен,
     `currentTitle`/`location` опциональны, `source` — один из `CandidateSource`, `tags` — массив строк
   - `CandidateForm.tsx` — один компонент на оба сценария (create/edit), принимает `defaultValues?`
     и `onSubmit`; `useForm({ resolver: yupResolver(schema), defaultValues })`; поля — обычные
     `<input>`/`<select>` через `register()` (не переусложнять `Controller` там, где не нужно —
     `Controller` требуется только для кастомных не-нативных инпутов, если такие сюда добавите позже)
   - Обязательно связать `<label htmlFor>` с `id` каждого инпута и вывести текст ошибки через
     `aria-describedby`, указывающий на элемент с сообщением об ошибке (`formState.errors.fullName?.message`)
     — это и есть тот самый a11y-минимум для форм, а не отдельная задача "на потом"
   - `useCreateCandidate.ts`/`useUpdateCandidate.ts` — мутации TanStack Query, на success —
     `queryClient.invalidateQueries(['candidates'])` и редирект на `/candidates/:id`
   - `pages/CandidateCreatePage/index.tsx` — рендерит `CandidateForm` без `defaultValues`, сабмит зовёт
     `useCreateCandidate`
   - `pages/CandidateEditPage/index.tsx` — грузит кандидата через `useCandidate(id)`, передаёт как
     `defaultValues`, сабмит зовёт `useUpdateCandidate`
   - Добавить роуты `/candidates/new` и `/candidates/:id/edit` в `app/routes.tsx`; на `CandidatesPage`
     — кнопка "Add candidate" (как в макете Employees list), на `CandidateProfilePage` — кнопка "Edit"

8. **Тесты (обязательно)**
   - `CandidateForm.test.tsx` — рендер формы, попытка сабмита с пустым `fullName` показывает текст
     ошибки валидации (RTL: `screen.getByText(...)`), заполнение валидных данных и сабмит вызывает
     `onSubmit` с ожидаемым объектом
   - Тест на `CandidatesFiltersStore` — юнит-тест MobX-стора **без рендера компонентов**: создать
     инстанс, вызвать `setSearch('x')`, проверить `store.search === 'x'` — MobX-сторы тестируются как
     обычные классы, DOM не нужен, это быстрее и проще RTL-тестов

## Критерии готовности

- [ ] Список кандидатов грузится, поиск по имени фильтрует список с debounce (не на каждый keystroke —
  запрос)
- [ ] Фильтры по статусу и источнику работают и комбинируются с поиском
- [ ] Клик по заголовку колонки сортирует
- [ ] Клик по строке открывает `/candidates/:id`
- [ ] На странице профиля переключение табов меняет контент без перезагрузки страницы
- [ ] Прямой переход по URL `/candidates/<конкретный-id>` открывает нужного кандидата (не только клик
  из списка)
- [ ] `/candidates/new` создаёт кандидата, после сабмита редиректит на его профиль, кандидат виден в списке
- [ ] `/candidates/:id/edit` предзаполнен текущими данными, сохранение обновляет профиль
- [ ] Валидация формы: пустой `fullName`/невалидный `email` блокирует сабмит и показывает текст ошибки,
  связанный с полем через `aria-describedby`
- [ ] `npx vitest run` зелёный: тесты на `CandidateForm` и `CandidatesFiltersStore` проходят

## Не делать сегодня

- Реальный upload/парсинг резюме
- Написание заметок (это День 6)
- Bulk-действия над выбранными чекбоксами (реализовать сам выбор строк можно, но действия над выбором —
  вне скоупа, если не останется свободного времени)
- Удаление кандидата — только create/edit
