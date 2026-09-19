# День 2 — Мок-данные, API-слой, Dashboard

## Цель дня

К концу дня: реальные (мок) данные текут через MSW → TanStack Query → Dashboard с 3–4 живыми виджетами
(не статичными картинками, а компонентами, которые примут любые данные).

## Пререквизиты

- День 1 завершён (layout, роутинг, тема)
- Прочитан `data-model.md`

## Файлы и папки к созданию

```
src/
  entities/
    candidate/
      model/types.ts
      mock/candidates.mock.ts
    job/
      model/types.ts
      mock/jobs.mock.ts
    stage/
      model/types.ts
      mock/stages.mock.ts
    pipeline/
      model/types.ts
      mock/pipelineEntries.mock.ts
    interview/
      model/types.ts
      mock/interviews.mock.ts
    note/
      model/types.ts
      mock/notes.mock.ts
    user/
      model/types.ts
      mock/users.mock.ts
  shared/
    api/
      client.ts                    (обёртка над fetch, base URL, error handling)
      mockServer/
        browser.ts                 (setupWorker из msw)
        handlers/
          candidates.handlers.ts
          jobs.handlers.ts
          pipeline.handlers.ts
          dashboard.handlers.ts    (агрегированные эндпойнты для метрик)
    lib/
      hooks/
        useDebouncedValue.ts       (пригодится позже для поиска)
  features/
    dashboard/
      api/
        useDashboardMetrics.ts     (TanStack Query хук)
      ui/
        StatTile/StatTile.tsx
        DonutChart/DonutChart.tsx
        BarChartComparison/BarChartComparison.tsx
        DashboardGrid/DashboardGrid.tsx
  shared/
    ui/
      Card/Card.tsx
  pages/
    DashboardPage/index.tsx        (обновить: собрать features/dashboard/ui в layout)
```

## Пошаговые задачи

1. **Типы сущностей**
   - Перенести интерфейсы из `data-model.md` в соответствующие `entities/*/model/types.ts` дословно —
     это тот случай, когда копирование спецификации в код оправдано

2. **Мок-данные**
   - Установить `@faker-js/faker`
   - `entities/user/mock/users.mock.ts` — 5 юзеров (написать первым, на него ссылаются остальные)
   - `entities/stage/mock/stages.mock.ts` — 6 фиксированных стадий (не через faker, руками: New,
     Screening, Interview, Offer, Hired, Rejected с `order` 0..5, `isTerminal: true` для Hired/Rejected)
   - `entities/job/mock/jobs.mock.ts` — 8–12 вакансий, `hiringManagerId` берёт случайного юзера с ролью
     `hiring_manager`
   - `entities/candidate/mock/candidates.mock.ts` — 40–60 кандидатов
   - `entities/pipeline/mock/pipelineEntries.mock.ts` — 60–100 записей, связывающих candidate↔job↔stage.
     **Важно**: распределение по стадиям должно быть неравномерным (больше в New/Screening, меньше в
     Hired) — иначе воронка на дашборде будет выглядеть как прямоугольник, а не воронка
   - `entities/interview/mock/interviews.mock.ts`, `entities/note/mock/notes.mock.ts` — по объёму из
     `data-model.md`

3. **API-слой (MSW)**
   - Установить `msw`, инициализировать `npx msw init public/ --save`
   - `shared/api/mockServer/handlers/*.handlers.ts` — REST-хендлеры поверх мок-массивов:
     - `GET /api/candidates` (с query-параметрами под будущие фильтры/поиск/пагинацию — заложить сразу)
     - `GET /api/jobs`
     - `GET /api/pipeline-entries?jobId=`
     - `GET /api/dashboard/metrics` — здесь считать агрегаты на лету из мок-массивов: total candidates,
       candidates by stage (для воронки), jobs by status, candidates by source
   - `shared/api/mockServer/browser.ts` — собрать `setupWorker(...handlers)`, запускать в `main.tsx`
     только в dev-режиме (`import.meta.env.DEV`)
   - `shared/api/client.ts` — тонкая обёртка `apiGet<T>(url)` на fetch с базовым error handling

4. **TanStack Query**
   - Установить `@tanstack/react-query`, обернуть `App` в `QueryClientProvider` (в `app/providers/`)
   - `features/dashboard/api/useDashboardMetrics.ts` — хук `useQuery(['dashboard-metrics'], ...)`

5. **UI-компоненты дашборда**
   - `shared/ui/Card/Card.tsx` — базовый контейнер-карточка (фон elevation-2, радиус, паддинг) —
     переиспользуется везде дальше, не только на дашборде
   - `features/dashboard/ui/StatTile` — пропсы: `label`, `value`, `deltaPercent`, `deltaDirection`
   - `features/dashboard/ui/DonutChart` — обёртка над `recharts` `PieChart`, по центру — крупное число
     (см. "556 employees" в макете) через абсолютно позиционированный div поверх SVG
   - `features/dashboard/ui/BarChartComparison` — `recharts` `BarChart` с двумя `<Bar>` (as in "Beginning
     of the year" / "End of the year")
   - `features/dashboard/ui/DashboardGrid` — CSS grid, собирает виджеты в layout, близкий к макету
     (2 колонки на верхнем ряду, широкие карточки ниже)

6. **Сборка страницы**
   - `pages/DashboardPage/index.tsx` — дергает `useDashboardMetrics`, обрабатывает `isLoading`/`isError`,
     рендерит `DashboardGrid` с реальными данными

7. **Тесты (обязательно)**
   - Установить `vitest`, `@testing-library/react`, `@testing-library/jest-dom`, `jsdom` — настроить
     `vite.config.ts`/`vitest.config.ts` один раз сегодня, дальше конфиг не трогать
   - `features/dashboard/ui/StatTile/StatTile.test.tsx` — рендер с разными `deltaDirection` (`up`/`down`),
     проверить, что цифра и знак дельты (+/-) отображаются корректно и с нужным цветовым классом
   - Один тест — не значит "любой ценой покрыть всё"; цель сегодня — обкатать сам сетап тестов на
     простом presentational-компоненте, дальше будет проще добавлять по одному тесту в день

## Критерии готовности

- [ ] Network-таб браузера показывает перехваченные MSW-запросы (не настоящую сеть)
- [ ] Dashboard показывает минимум: total candidates (StatTile), candidates by stage (donut или bar —
  воронка), jobs by status, один сравнительный bar chart
- [ ] При изменении мок-данных (добавить кандидата в массив) цифры на дашборде меняются после перезагрузки
- [ ] Loading state виден на секунду при первой загрузке (можно искусственно добавить задержку в MSW
  handler через `delay()` из msw, чтобы состояние вообще было видно)
- [ ] `npx vitest run` проходит зелёным, есть минимум 1 тест на `StatTile`

## Не делать сегодня

- Candidates list/Jobs list как полноценные страницы — сегодня только агрегаты для дашборда
- Фильтры дашборда по датам (в макете есть "Last year" — красиво, но не критично для MVP, можно вернуться
  в День 7 если останется время)
