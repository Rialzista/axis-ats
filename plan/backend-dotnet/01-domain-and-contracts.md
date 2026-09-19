# Domain & Contracts — сквозная спецификация (бэкенд)

Контракт типов, не реализация — как и фронтенд `data-model.md`. Сущности кладёте в
`Ats.Domain/Entities/*.cs`, DTO — в `Ats.Application/<Feature>/Dto/*.cs`.

## Соглашение по именованию

- Domain-сущности: PascalCase, без суффиксов (`Candidate`, не `CandidateEntity`)
- DTO для ответа: суффикс `Dto` (`CandidateDto`)
- DTO для запроса/создания: суффикс `Request` (`CreateNoteRequest`)
- JSON на границе API — **camelCase** (фронтенд ждёт `fullName`, не `FullName`) — настраивается один
  раз глобально в `Program.cs` через `JsonSerializerOptions.PropertyNamingPolicy`, не руками на каждом DTO

## Candidate

```csharp
// Ats.Domain/Entities/Candidate.cs
public class Candidate
{
    public Guid Id { get; set; }
    public string FullName { get; set; } = default!;
    public string? AvatarUrl { get; set; }
    public string Email { get; set; } = default!;
    public string? Phone { get; set; }
    public string? Location { get; set; }
    public string? CurrentTitle { get; set; }
    public CandidateSource Source { get; set; }
    public CandidateStatus Status { get; set; }
    public string? ResumeUrl { get; set; }
    public List<string> Tags { get; set; } = new();       // Postgres: jsonb или text[]
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }

    public List<PipelineEntry> PipelineEntries { get; set; } = new();
    public List<Note> Notes { get; set; } = new();
    public List<ActivityEvent> ActivityEvents { get; set; } = new();
}

public enum CandidateSource { LinkedIn, Referral, Website, Agency, Other }
public enum CandidateStatus { Active, Hired, Rejected, OnHold }
```

DTO: `CandidateListItemDto` (укороченный, для таблицы — без всех полей) и `CandidateDetailsDto`
(полный, для профиля) — **не один общий DTO на всё**, список и деталь показывают разный набор полей,
это осознанное разделение, а не дублирование.

## Job

```csharp
public class Job
{
    public Guid Id { get; set; }
    public string Title { get; set; } = default!;
    public string Department { get; set; } = default!;
    public string Location { get; set; } = default!;
    public EmploymentType EmploymentType { get; set; }
    public JobStatus Status { get; set; }
    public Guid HiringManagerId { get; set; }
    public User HiringManager { get; set; } = default!;
    public DateTime OpenedAt { get; set; }
    public DateTime? ClosedAt { get; set; }
    public string Description { get; set; } = default!;

    public List<PipelineEntry> PipelineEntries { get; set; } = new();
}

public enum EmploymentType { FullTime, PartTime, Contract }
public enum JobStatus { Open, OnHold, Closed }
```

## Stage

Стадии — справочник, сидится один раз при старте (миграция + сид-данные), не редактируется через API
в рамках этой недели.

```csharp
public class Stage
{
    public Guid Id { get; set; }
    public string Name { get; set; } = default!;
    public int Order { get; set; }
    public bool IsTerminal { get; set; }
}
```

## PipelineEntry

```csharp
public class PipelineEntry
{
    public Guid Id { get; set; }
    public Guid CandidateId { get; set; }
    public Candidate Candidate { get; set; } = default!;
    public Guid JobId { get; set; }
    public Job Job { get; set; } = default!;
    public Guid StageId { get; set; }
    public Stage Stage { get; set; } = default!;
    public DateTime MovedAt { get; set; }
    public string? RejectionReason { get; set; }

    public List<Interview> Interviews { get; set; } = new();
}
```

## Interview

```csharp
public class Interview
{
    public Guid Id { get; set; }
    public Guid PipelineEntryId { get; set; }
    public PipelineEntry PipelineEntry { get; set; } = default!;
    public DateTime ScheduledAt { get; set; }
    public List<Guid> InterviewerIds { get; set; } = new();   // jsonb
    public InterviewType Type { get; set; }

    // Scorecard — value object, хранить как jsonb-колонку через EF Core owned type
    // или через .HasConversion на JSON-сериализацию — не отдельная таблица, это не оправдано объёмом
    public Scorecard? Scorecard { get; set; }
}

public enum InterviewType { PhoneScreen, Technical, Onsite, Final }

public class Scorecard
{
    public int OverallRating { get; set; }          // 1..5
    public List<ScorecardCriterion> Criteria { get; set; } = new();
    public Recommendation Recommendation { get; set; }
}

public class ScorecardCriterion
{
    public string Name { get; set; } = default!;
    public int Rating { get; set; }
    public string? Comment { get; set; }
}

public enum Recommendation { StrongYes, Yes, No, StrongNo }
```

## Note & ActivityEvent

```csharp
public class Note
{
    public Guid Id { get; set; }
    public Guid CandidateId { get; set; }
    public Guid AuthorId { get; set; }
    public string Text { get; set; } = default!;
    public DateTime CreatedAt { get; set; }
}

public class ActivityEvent
{
    public Guid Id { get; set; }
    public Guid CandidateId { get; set; }
    public ActivityEventType Type { get; set; }
    public string PayloadJson { get; set; } = default!;   // jsonb: { "from": "...", "to": "..." } и т.п.
    public Guid ActorId { get; set; }
    public DateTime CreatedAt { get; set; }
}

public enum ActivityEventType { StageChanged, InterviewScheduled, FieldUpdated, NoteAdded }
```

## User

```csharp
public class User
{
    public Guid Id { get; set; }
    public string FullName { get; set; } = default!;
    public string? AvatarUrl { get; set; }
    public UserRole Role { get; set; }
}

public enum UserRole { Recruiter, HiringManager, Admin }
```

## DTO-слой — что реально уходит во фронтенд

Держите enum'ы на границе API как **строки** (`"active"`, `"stage_changed"`), не как числа — фронтенд
уже написан под строковые union-типы из `data-model.md` (`CandidateStatus = 'active' | ...`). В .NET
это включается через `JsonStringEnumConverter` в опциях сериализации — иначе придётся переписывать
типы на фронте под числа, а это уже не "бэкенд не трогает фронт".

Аналогично **camelCase** для enum-значений: `StageChanged` → `"stage_changed"` — стандартный
`JsonStringEnumConverter` даёт PascalCase, для snake_case понадобится либо кастомный конвертер, либо
согласовать с фронтом camelCase enum'ы (`"stageChanged"`) и на Дне 8 поправить это в двух местах на
фронте (`entities/*/model/types.ts`). Примите решение осознанно на Дне 8, а не по ходу дела.

## Query-параметры списков (Candidates/Jobs)

Общий паттерн для `GET /api/candidates` и `GET /api/jobs` — не изобретать два разных подхода:

```csharp
public class CandidatesQuery
{
    public string? Search { get; set; }
    public CandidateStatus? Status { get; set; }
    public CandidateSource? Source { get; set; }
    public string? SortBy { get; set; }      // "fullName" | "createdAt" | ...
    public bool SortDesc { get; set; }
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 25;
}

public class PagedResult<T>
{
    public List<T> Items { get; set; } = new();
    public int TotalCount { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
}
```

Если фронтенд-хук `useCandidatesList` в Фазе 1 писался без пагинации (просто возвращал весь массив) —
на Дне 10 либо доработайте фронт под `PagedResult<T>`, либо сознательно упростите бэкенд-ответ до
плоского массива без пагинации. Зафиксируйте выбор перед началом Дня 10, не решайте на ходу.
