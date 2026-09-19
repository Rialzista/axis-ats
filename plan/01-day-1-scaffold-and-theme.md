# День 1 — Скаффолд, тема, layout

## Цель дня

К концу дня: пустое, но полностью навигируемое приложение — тёмная тема в стиле Axis HRM,
Sidebar с пунктами меню, Topbar, роутинг между заглушками страниц.

## Пререквизиты

- Прочитан `design-tokens.md`
- Прочитан `data-model.md` (пока не нужен для кода, но держите в голове структуру перед Днём 2)
- Пройден `setup.md` (софт установлен)

## Шаг 0 — Git репозиторий

Сделать **до** `npm create vite` — иначе первый коммит рискует случайно не включить что-то из уже
сделанного руками, либо про него просто забудут после того, как проект заработает.

- [ ] `git init` в корне `axis-ats/` (если ещё не инициализирован)
- [ ] `.gitignore` создан (node_modules, dist, .env*, .DS_Store — для .NET-части ещё понадобится
  дополнить в День 8: `bin/`, `obj/`, `appsettings.Development.json`)
- [ ] Первый коммит: `git add plan/ .gitignore && git commit -m "docs: project plan"` — план должен
  попасть в историю первым же коммитом, до кода
- [ ] Репозиторий создан на GitHub, `git remote add origin <url>` и `git push -u origin main` — не
  откладывать до Дня 7, чтобы не потерять историю локальных коммитов при проблемах с диском/машиной

## Файлы и папки к созданию

```
axis-ats/
  index.html
  vite.config.ts
  tailwind.config.ts
  tsconfig.json
  package.json
  src/
    main.tsx
    app/
      App.tsx
      routes.tsx
      layouts/
        AppLayout/
          AppLayout.tsx
          AppLayout.module.css        (или tailwind-классы, на ваш выбор)
    pages/
      DashboardPage/index.tsx
      CandidatesPage/index.tsx
      CandidateProfilePage/index.tsx
      JobsPage/index.tsx
      JobDetailPage/index.tsx
      PipelinePage/index.tsx
      ReportsPage/index.tsx
    shared/
      ui/
        Sidebar/
          Sidebar.tsx
          Sidebar.types.ts
        Topbar/
          Topbar.tsx
        Icon/
          Icon.tsx                    (обёртка над выбранной иконочной библиотекой)
      styles/
        tokens.css
        fonts.css
        globals.css
      config/
        routes.ts                    (enum/const путей, чтобы не хардкодить строки)
        navigation.ts                (массив пунктов меню: label, icon, path)
```

## Пошаговые задачи

1. **Инициализация проекта**
   - `npm create vite@latest . -- --template react-ts`
   - Установить зависимости: `react-router-dom`, `tailwindcss` (+ postcss, autoprefixer), `clsx`
   - Настроить `tailwind.config.ts`: подключить шрифты и цвета из `design-tokens.md` через `theme.extend.colors`
     и `theme.extend.fontFamily` — **не хардкодить hex прямо в компонентах**

2. **Design tokens**
   - `src/shared/styles/tokens.css` — CSS custom properties из `design-tokens.md` (`--color-*`, `--font-*`)
   - `src/shared/styles/fonts.css` — `@import` шрифтов с Google Fonts (JetBrains Mono + Inter)
   - `src/shared/styles/globals.css` — сброс базовых стилей, `body { background: var(--color-bg-elevation-1) }`
   - Подключить оба файла в `main.tsx`

3. **Роутинг**
   - `src/shared/config/routes.ts` — константы путей: `/`, `/candidates`, `/candidates/:id`, `/jobs`,
     `/jobs/:id`, `/pipeline`, `/reports`
   - `src/app/routes.tsx` — конфиг React Router (`createBrowserRouter` или `<Routes>`), каждая страница
     пока рендерит просто `<h1>{PageName}</h1>`
   - `src/app/App.tsx` — оборачивает роутинг в `AppLayout`

4. **AppLayout**
   - Grid/flex: слева `Sidebar` (фикс. ширина ~220px), сверху `Topbar`, справа `<Outlet />` со скроллом
   - На этом шаге просто структура, без сложной адаптивности (mobile — можно отложить на День 7)

5. **Sidebar**
   - `src/shared/config/navigation.ts` — массив `{ label, path, icon }` для: Dashboard, Candidates, Jobs,
     Pipeline, Reports
   - `Sidebar.tsx` рендерит логотип/название проекта сверху + список пунктов из `navigation.ts`,
     активный пункт подсвечивается через `NavLink` (React Router сам даёт `.active` класс)
   - Задел на будущее: пункт может иметь необязательный counter-badge (как "Mana 3" в макете) —
     не реализовывать сейчас, просто не блокировать тип от этого поля в `Sidebar.types.ts`

6. **Topbar**
   - Простая полоса сверху: название текущей страницы (можно взять из `navigation.ts` по текущему path)
     + место под будущий поиск/аватар (пока просто пустой div-плейсхолдер)

7. **Проверка глазами**
   - Открыть каждый пункт меню руками в браузере, убедиться что роутинг работает и тема (тёмный фон,
     шрифты) применена глобально, а не только на одной странице

## Критерии готовности

- [ ] `npm run dev` поднимается без ошибок
- [ ] Все 5 пунктов меню кликабельны и меняют URL + контент
- [ ] Активный пункт меню визуально выделен акцентным цветом
- [ ] Цвета/шрифты берутся из CSS-переменных/Tailwind-темы, а не из инлайн-хардкода
- [ ] Нет ни одного `console.error` в браузерной консоли

## Не делать сегодня

- Реальные данные, API, MSW — рано
- Адаптивная вёрстка под мобильные — только desktop-first
- Тесты
- Детальная проработка иконок (можно взять любую иконочную библиотеку типа `lucide-react` и не тратить
  время на кастомные SVG)
