# День 4 — Jobs: список, карточка вакансии, создание/редактирование

## Цель дня

`JobsPage` (список вакансий, аналогично Candidates list), `JobDetailPage` (карточка вакансии со
связанными кандидатами по стадиям — подготовка почвы под Kanban в День 5), и форма создания/
редактирования вакансии на React Hook Form + Yup (тот же паттерн, что и Candidate form в Дне 3).

> Как и День 3 — это, вероятно, полтора-два вечера с добавленной формой. Пункты 1–5 (список/деталь)
> и пункт 6 (форма) можно разносить по сессиям.

## Пререквизиты

- День 3 завершён (DataTable, Tabs, Badge, Avatar, MobX-паттерн стора, RHF+Yup паттерн формы уже обкатаны)

## Файлы и папки к созданию

```
src/
  features/
    jobs/
      api/
        useJobsList.ts
        useJob.ts
        useJobPipelineEntries.ts     (кандидаты конкретной вакансии со стадиями)
        useCreateJob.ts
        useUpdateJob.ts
      model/
        JobsFiltersStore.ts          (MobX-класс, по образцу CandidatesFiltersStore из Дня 3)
        jobFormSchema.ts             (Yup-схема)
      ui/
        JobsTable/
          JobsTable.tsx
          columns.tsx
        JobsFilters/JobsFilters.tsx
        JobCard/JobCard.tsx          (шапка карточки вакансии: title, department, status, hiring manager)
        JobCandidatesByStage/
          JobCandidatesByStage.tsx   (пока просто сгруппированный список, не drag&drop — это День 5)
        JobForm/
          JobForm.tsx                (RHF-форма, общая для create и edit)
          JobForm.test.tsx
  shared/
    ui/
      StatusPill/StatusPill.tsx      (цветной индикатор статуса: open/on_hold/closed — переиспользуется
                                       и для Job.status, и позже можно для Candidate.status)
  pages/
    JobsPage/index.tsx
    JobDetailPage/index.tsx
    JobCreatePage/index.tsx
    JobEditPage/index.tsx
```

## Пошаговые задачи

1. **Jobs list**
   - Аналогично Дню 3: `JobsFiltersStore.ts` (search, department, status — тот же MobX-паттерн
     `makeAutoObservable`), `useJobsList` через MSW handler `GET /api/jobs` (расширить handler из
     Дня 2 фильтрацией/сортировкой)
   - `columns.tsx`: Title, Department, Location, Employment type, Status (`StatusPill`), Hiring manager
     (Avatar+имя — нужен join с `User` на клиенте или на стороне MSW handler, решите сами что проще),
     Open date, кол-во кандидатов в пайплайне (посчитать через `PipelineEntry`)
   - `JobsFilters.tsx` + `JobsTable.tsx` + `pages/JobsPage/index.tsx` — по образцу Candidates

2. **StatusPill**
   - Небольшой компонент: цветная точка/пилюля + текст, цвет зависит от статуса (open — акцент/зелёный,
     on_hold — жёлтый, closed — серый). Сделать generic по `status: string` + `colorMap: Record<string,
     string>`, чтобы переиспользовать для разных enum'ов домена, а не только Job.status

3. **Job detail — шапка**
   - `useJob(id)` — данные вакансии
   - `JobCard.tsx` — title, department, location, employment type, status, hiring manager (Avatar+имя,
     кликабельно — по желанию можно не делать ссылку на профиль юзера, т.к. страницы юзеров вне скоупа),
     description (текстовый блок)

4. **Job detail — кандидаты по стадиям**
   - `useJobPipelineEntries(jobId)` — все `PipelineEntry` этой вакансии, с джойном на `Candidate` и `Stage`
   - `JobCandidatesByStage.tsx` — сгруппировать `PipelineEntry` по `stageId`, вывести как список колонок
     (визуально уже похоже на будущий Kanban, но **без drag&drop** — просто `<div>` колонки со списком
     карточек кандидатов). Это сознательный промежуточный шаг: сначала данные и группировка, drag&drop
     добавляется поверх готовой структуры в День 5, а не одновременно с ней

5. **Роутинг**
   - `pages/JobDetailPage/index.tsx` — `:id` из `useParams`, собирает `JobCard` + `JobCandidatesByStage`

6. **Create/Edit Job — форма (RHF + Yup)**
   - `features/jobs/model/jobFormSchema.ts` — Yup-схема: `title`/`department`/`location` required,
     `employmentType`/`status` — один из соответствующих enum'ов, `description` required (минимальная
     длина, например 20 символов — пустое описание вакансии бессмысленно), `hiringManagerId` required
     (select из списка `User` с ролью `hiring_manager`)
   - `JobForm.tsx` — по образцу `CandidateForm` из Дня 3: один компонент на create/edit,
     `useForm({ resolver: yupResolver(schema) })`, `<label htmlFor>` + `aria-describedby` на ошибках —
     тот же a11y-минимум, не повторяю то же самое, что делали вчера, но не пропускаю
   - `useCreateJob.ts`/`useUpdateJob.ts` — мутации, инвалидация `['jobs']`, редирект на `/jobs/:id`
   - `pages/JobCreatePage/index.tsx`, `pages/JobEditPage/index.tsx`, роуты `/jobs/new`, `/jobs/:id/edit`,
     кнопки "Add job"/"Edit" в соответствующих местах UI

7. **Тесты (обязательно)**
   - `JobForm.test.tsx` — по образцу `CandidateForm.test.tsx`: пустой `title` блокирует сабмит с текстом
     ошибки, валидные данные сабмитятся с ожидаемым объектом
   - Тест на `StatusPill` — рендер с разными статусами даёт разный текст/цветовой класс (простой
     presentational-тест, хороший кандидат на первую историю в Storybook позже, в Дне 7)

## Критерии готовности

- [ ] Jobs list грузится, фильтруется, сортируется — по аналогии с Candidates
- [ ] Клик по строке ведёт на `/jobs/:id`
- [ ] На странице вакансии видно кандидатов, сгруппированных по стадиям, в виде колонок
- [ ] Счётчик кандидатов в колонке Jobs list совпадает с реальным числом `PipelineEntry` для этой вакансии
  (проверить руками на 2–3 вакансиях)
- [ ] `/jobs/new` создаёт вакансию, `/jobs/:id/edit` редактирует существующую, оба валидируют форму
- [ ] `npx vitest run` зелёный: тесты на `JobForm` и `StatusPill` проходят

## Не делать сегодня

- Drag&drop — завтра
- Удаление вакансии — только create/edit, как и с Candidate в Дне 3
- Переход из карточки кандидата в стадии обратно на вакансию — не обязательно, если не останется времени
