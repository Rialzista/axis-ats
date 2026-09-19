# День 17 (бонус) — Pipeline как Module Federation remote

## Цель дня

Host (основное ATS-приложение) динамически загружает Pipeline-фичу из отдельно собранного и
задеплоенного remote-приложения через `@module-federation/vite`.

## Пререквизиты

- Фазы 1–3 полностью завершены и стабильны
- Прочитан `00-overview.md` этой фазы, архитектурное решение (монорепо с двумя Vite-проектами) принято

## Файлы и папки к созданию

```
axis-ats/
  vite.config.ts                      (обновить: federation plugin, remotes)
  src/
    app/
      remotes/
        PipelineRemote.tsx            (обёртка: React.lazy + Suspense вокруг remote-модуля)
pipeline-remote/
  package.json
  vite.config.ts                      (federation plugin, exposes)
  index.html
  src/
    main.tsx                          (standalone entry — для разработки remote в изоляции)
    PipelineApp.tsx                   (экспортируемый компонент — точка входа для хоста)
```

## Пошаговые задачи

1. **Remote-проект**
   - `npm create vite@latest pipeline-remote -- --template react-ts` рядом с основным проектом
   - Установить те же версии `react`/`react-dom`, что в хосте (свериться с `axis-ats/package.json`
     — версии обязаны совпадать для `shared` в федерации)
   - Скопировать/перенести `features/pipeline/*` и связанные зависимости (`KanbanBoard`, хуки на
     TanStack Query, MobX-сторы, SignalR-подключение из Фазы 3) — на первой итерации проще
     скопировать, чем городить общий npm-пакет между host и remote ради проекта

2. **`@module-federation/vite` — remote**
   - `npm install @module-federation/vite` в `pipeline-remote`
   - В `vite.config.ts` remote: `federation({ name: 'pipeline', filename: 'remoteEntry.js', exposes: {
     './PipelineApp': './src/PipelineApp.tsx' }, shared: ['react', 'react-dom', '@tanstack/react-query',
     'mobx', 'mobx-react-lite'] })`
   - `PipelineApp.tsx` — принимает `jobId` пропом (не читает роутер напрямую — роутинг остаётся в
     хосте), рендерит `KanbanBoard`

3. **`@module-federation/vite` — host**
   - `npm install @module-federation/vite` в `axis-ats`
   - В `vite.config.ts` хоста: `federation({ name: 'host', remotes: { pipeline: 'http://localhost:<port-remote>/assets/remoteEntry.js' },
     shared: [...тот же список] })`
   - `src/app/remotes/PipelineRemote.tsx`: `const PipelineApp = React.lazy(() => import('pipeline/PipelineApp'))`,
     обернуть в `<Suspense fallback={<Skeleton />}>`
   - В `PipelinePage`/`JobDetailPage` (Фаза 1, Дни 4–5) заменить прямой импорт `KanbanBoard` на
     `PipelineRemote` — это единственное место в существующем коде, которое меняется

4. **Dev-режим — два сервера одновременно**
   - `pipeline-remote`: `npm run dev` (например, порт 5001, remote должен быть собран в `preview`-режиме
     для федерации — dev-режим Vite не всегда отдаёт `remoteEntry.js` так же, как prod-сборка;
     проверить, требует ли `@module-federation/vite` `vite build && vite preview` для remote вместо
     обычного dev-сервера — если да, зафиксировать это как рабочий процесс, не бороться с ним)
   - `axis-ats` (хост): `npm run dev` на своём порту как обычно
   - Задокументировать это в `pipeline-remote/README.md` — два отдельных процесса не запускаются одной
     командой без дополнительной оснастки (`concurrently`/`npm-run-all`), можно добавить, если хочется

5. **Деплой remote отдельно от хоста**
   - Задеплоить `pipeline-remote` как отдельный статический сайт (Vercel/Netlify — так же, как хост)
   - Обновить `remotes` в `vite.config.ts` хоста на прод-URL remote вместо `localhost`
   - Пересобрать и передеплоить хост

## Критерии готовности

- [ ] `pipeline-remote` собирается независимо (`npm run build`), отдаёт `remoteEntry.js`
- [ ] Хост в dev-режиме подгружает Kanban из remote (проверить в Network — запрос за `remoteEntry.js`
  на порт remote, не в бандле хоста)
- [ ] React не задублирован в финальных бандлах (DevTools → Sources, либо `webpack-bundle-analyzer`-
  аналог для Vite, например `rollup-plugin-visualizer`, на обоих проектах)
- [ ] Drag&drop и realtime (SignalR) в Kanban продолжают работать без изменений внутри remote
- [ ] Изменение верстки внутри `pipeline-remote` и его пересборка видны в хосте после обновления
  страницы, **без** пересборки хоста — зафиксировать это на скриншоте/gif для демонстрации на
  собеседовании, это и есть весь смысл дня

## Не делать сегодня

- Вынос других фич (Candidates, Jobs) в remote — Pipeline единственный кандидат, дальше не масштабируем
- CI/CD с независимым деплоем host/remote по pull request — это отдельная (интересная) задача, но не
  на сегодня
- Version negotiation между несколькими версиями remote одновременно — сценарий из реальных
  микрофронтенд-платформ с несколькими командами, не имеет смысла для проекта с одним автором
