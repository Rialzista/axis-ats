# День 10 — Candidates API + первое подключение фронтенда

## Цель дня

Полностью рабочие эндпоинты Candidates (список с фильтрами/поиском/сортировкой, деталь, notes,
activity), плюс фронтенд Дня 3 (Фаза 1) переключён на реальный бэкенд вместо MSW — чтобы не копить
интеграционный риск до последнего дня.

## Пререквизиты

- День 9 завершён (БД с данными)
- Фронтенд Фазы 1, Дни 1–3 реализованы

## Файлы и папки к созданию

```
src/
  Ats.Application/
    Candidates/
      Dto/
        CandidateListItemDto.cs
        CandidateDetailsDto.cs
        NoteDto.cs
        CreateNoteRequest.cs
        ActivityEventDto.cs
      CandidatesQuery.cs
      ICandidatesService.cs
      CandidatesService.cs
      CandidateMappingExtensions.cs        (ToListItemDto / ToDetailsDto)
  Ats.Infrastructure/
    Repositories/
      ICandidatesRepository.cs             (опционально — см. пункт 2)
      CandidatesRepository.cs
  Ats.Api/
    Endpoints/
      CandidatesEndpoints.cs
    Common/
      PagedResult.cs
```

## Пошаговые задачи

1. **DTO**
   - `CandidateListItemDto`: id, fullName, avatarUrl, email, currentTitle, status, source, tags,
     createdAt — то, что реально рендерится в колонках таблицы (Фаза 1, День 3, `columns.tsx`)
   - `CandidateDetailsDto`: все поля + `appliedJobIds` — для `CandidateCard`/`InfoTab`
   - `NoteDto`, `CreateNoteRequest` (`{ text: string, authorId: Guid }`), `ActivityEventDto`
     (`id, type, payload (object, не строка — десериализовать PayloadJson перед отдачей), actorId,
     createdAt`)

2. **Слой доступа к данным — решение по архитектуре**
   - Для проекта такого объёма **репозиторий поверх DbContext — опционально**, можно обращаться к
     `AtsDbContext` напрямую из `CandidatesService`, если находите отдельный интерфейс избыточным
     ритуалом. Если хотите потренировать паттерн Repository отдельно от EF Core — заведите
     `ICandidatesRepository`/`CandidatesRepository`. Примите решение осознанно и держите единообразно
     по всем фичам (Candidates/Jobs/Pipeline) — не мешайте подходы между фичами

3. **CandidatesService**
   - `GetListAsync(CandidatesQuery query)` → `PagedResult<CandidateListItemDto>` (или плоский список —
     решение зафиксировано на Дне 8/9, см. `01-domain-and-contracts.md`): `Where` по `Search` (ILIKE по
     `FullName`/`Email`), по `Status`/`Source` если заданы, `OrderBy`/`OrderByDescending` по `SortBy`,
     `Skip`/`Take` для пагинации
   - `GetByIdAsync(Guid id)` → `CandidateDetailsDto?`
   - `GetNotesAsync(Guid candidateId)`, `AddNoteAsync(Guid candidateId, CreateNoteRequest request)` —
     при добавлении заметки **также создавать `ActivityEvent` с `Type = NoteAdded`** (та же логика,
     что на фронте предполагалась в Фазе 1 День 6 — здесь она переезжает на бэкенд как источник истины)
   - `GetActivityAsync(Guid candidateId)` → объединённый список Notes + ActivityEvents, отсортированный
     по `CreatedAt desc` — можно двумя запросами и склеиванием в памяти, для этого объёма данных не
     проблема с производительностью

4. **Endpoints (Minimal API)**
   - `CandidatesEndpoints.cs`, extension `MapCandidatesEndpoints(this WebApplication app)`:
     - `GET /api/candidates` — биндинг query-параметров в `CandidatesQuery` (`[AsParameters]`)
     - `GET /api/candidates/{id:guid}` — 404 если не найден
     - `GET /api/candidates/{id:guid}/notes`
     - `POST /api/candidates/{id:guid}/notes` — валидация через FluentValidation (см. День 13, но
       базовую валидацию `Text` не пустой можно и сегодня через `if`/`Results.BadRequest`, не
       обязательно ждать День 13 ради одной проверки)
     - `GET /api/candidates/{id:guid}/activity`
   - Регистрация в `Program.cs`: `app.MapCandidatesEndpoints();`

5. **Подключение фронтенда**
   - В `shared/api/client.ts` (Фаза 1) сменить базовый URL на `http://localhost:5000/api` (или порт,
     на котором слушает .NET)
   - Отключить инициализацию MSW для Candidates-related хендлеров — либо полностью выключить MSW worker
     на этом этапе (раз всё равно к концу Фазы 2 он не нужен), либо оставить для Jobs/Pipeline, которые
     ещё не портированы (решить, что проще — вероятно, проще выключить совсем и на Днях 11–12 просто
     не трогать фронт до готовности соответствующих бэкенд-эндпоинтов)
   - Прогнать вручную: список кандидатов, поиск, фильтры, открытие профиля, добавление заметки —
     всё должно работать один в один как на MSW, но данные теперь из Postgres

## Критерии готовности

- [ ] Все эндпоинты Candidates видны и вызываемы через Swagger
- [ ] Фронтенд-страницы Candidates list и Candidate profile работают против реального бэкенда
- [ ] Добавленная через UI заметка видна в Adminer в таблице `notes` и в ленте активности после
  обновления страницы (переживает рестарт фронтенда — подтверждение, что данные реально в БД, а не в
  памяти процесса)
- [ ] Поиск/фильтры/сортировка на фронте дают тот же результат, что и раньше на MSW (пройтись по тем
  же ручным сценариям, что использовали в Фазе 1 День 3)

## Не делать сегодня

- Jobs/Pipeline/Interviews эндпоинты — завтра
- CORS для прод-домена — пока только localhost
