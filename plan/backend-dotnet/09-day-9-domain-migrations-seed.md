# День 9 — Доменные сущности, миграции, сидинг

## Цель дня

Все сущности из `01-domain-and-contracts.md` описаны как EF-сущности, миграция создаёт схему в
Postgres, база наполнена сид-данными сопоставимого объёма с фронтенд-моками.

## Пререквизиты

- День 8 завершён (DbContext подключается, health-check работает)

## Файлы и папки к созданию

```
src/
  Ats.Domain/
    Entities/
      Candidate.cs
      Job.cs
      Stage.cs
      PipelineEntry.cs
      Interview.cs
      Scorecard.cs
      Note.cs
      ActivityEvent.cs
      User.cs
    Enums/
      CandidateSource.cs
      CandidateStatus.cs
      EmploymentType.cs
      JobStatus.cs
      InterviewType.cs
      Recommendation.cs
      ActivityEventType.cs
      UserRole.cs
  Ats.Infrastructure/
    Persistence/
      AtsDbContext.cs                    (обновить: добавить DbSet<T> на все сущности)
      Configurations/
        CandidateConfiguration.cs
        JobConfiguration.cs
        StageConfiguration.cs
        PipelineEntryConfiguration.cs
        InterviewConfiguration.cs
        NoteConfiguration.cs
        ActivityEventConfiguration.cs
        UserConfiguration.cs
    Seed/
      SeedData.cs
      DatabaseSeeder.cs
```

## Пошаговые задачи

1. **Сущности и enum'ы**
   - Перенести классы дословно из `01-domain-and-contracts.md` в `Ats.Domain/Entities/*.cs` и
     `Ats.Domain/Enums/*.cs` — как и на фронте, копирование спецификации в код здесь оправдано

2. **Fluent API конфигурации**
   - Для каждой сущности — отдельный `IEntityTypeConfiguration<T>` в `Configurations/` (не сваливать
     всё в `OnModelCreating` одним большим методом — плохо читается и плохо масштабируется)
   - Ключевые моменты конфигурации:
     - `Candidate.Tags` → `List<string>` замаппить на `jsonb`-колонку (`.HasColumnType("jsonb")` +
       value comparer для EF, чтобы отслеживал изменения внутри списка)
     - `Interview.InterviewerIds` → аналогично `jsonb`
     - `Interview.Scorecard` → owned type (`.OwnsOne(i => i.Scorecard, ...)`) либо сериализация в
       `jsonb` через `.HasConversion` — выбрать один подход и не смешивать паттерны между сущностями
     - `ActivityEvent.PayloadJson` → обычная `text`/`jsonb` колонка, без owned type (форма payload
       разная в зависимости от `Type`, строго типизировать не нужно)
     - Индексы: `Candidate.Email` (unique), `PipelineEntry(CandidateId, JobId)` (composite, не
       обязательно unique — кандидат в теории может быть дважды в одной вакансии в разных циклах, но
       для проекта можно сделать unique и явно этим ограничить домен)
   - Применить все конфигурации разом в `AtsDbContext.OnModelCreating` через
     `modelBuilder.ApplyConfigurationsFromAssembly(typeof(AtsDbContext).Assembly)`

3. **DbSet'ы**
   - Добавить `DbSet<Candidate> Candidates`, `DbSet<Job> Jobs`, `DbSet<Stage> Stages`,
     `DbSet<PipelineEntry> PipelineEntries`, `DbSet<Interview> Interviews`, `DbSet<Note> Notes`,
     `DbSet<ActivityEvent> ActivityEvents`, `DbSet<User> Users` в `AtsDbContext`

4. **Миграция**
   - `dotnet ef migrations add InitialCreate --project src/Ats.Infrastructure --startup-project
     src/Ats.Api`
   - Посмотреть сгенерированный SQL глазами (`dotnet ef migrations script`) — на проекте это
     хорошая привычка, не слепо доверять генератору
   - `dotnet ef database update --project src/Ats.Infrastructure --startup-project src/Ats.Api`
   - Проверить через Adminer (из Дня 8), что таблицы появились с ожидаемыми колонками и типами

5. **Сидинг**
   - Установить `Bogus` (аналог `@faker-js/faker` для .NET) в `Ats.Infrastructure`
   - `Seed/SeedData.cs` — статические/сгенерированные наборы: 5 `User`, 6 `Stage` (руками, не через
     Bogus — фиксированный список New/Screening/Interview/Offer/Hired/Rejected с `Order` 0..5),
     8–12 `Job`, 40–60 `Candidate`, 60–100 `PipelineEntry` (то же требование неравномерного
     распределения по стадиям, что и во фронтенд-моках — иначе воронка на дашборде будет прямоугольником),
     15–20 `Interview`, 30–50 `Note`/`ActivityEvent`
   - `Seed/DatabaseSeeder.cs` — метод `SeedAsync(AtsDbContext context)`: если `context.Users.Any()` —
     выйти (не дублировать сид при каждом рестарте), иначе — вставить данные в правильном порядке
     (Users/Stages → Jobs → Candidates → PipelineEntries → Interviews/Notes/ActivityEvents, т.к. есть
     foreign key зависимости)
   - Вызвать сидер в `Program.cs` при старте в Development-окружении (создать scope, получить
     `AtsDbContext`, вызвать `SeedAsync`)

## Критерии готовности

- [ ] `dotnet ef database update` проходит без ошибок на чистой БД
- [ ] После первого запуска приложения в БД видно наполненные таблицы (проверить в Adminer)
- [ ] Повторный запуск приложения не плодит дубликаты (сидер идемпотентен)
- [ ] Распределение `PipelineEntry` по стадиям неравномерное — руками посчитать через SQL-запрос
  в Adminer (`SELECT stage_id, count(*) FROM pipeline_entries GROUP BY stage_id`) и убедиться, что
  ранние стадии заметно многолюднее поздних

## Не делать сегодня

- Ни одного HTTP-эндпоинта — сегодня только данные и схема
- Оптимизация индексов под нагрузку — преждевременно для объёма данных проекта
