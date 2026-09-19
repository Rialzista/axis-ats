# Data Model — сквозная спецификация сущностей

Это контракт типов, не реализация. Кладёте в `src/entities/<entity>/model/types.ts` каждый блок,
реализацию мок-генераторов пишете сами в `src/entities/<entity>/mock/`.

## Candidate (`entities/candidate`)

```ts
interface Candidate {
  id: string;
  fullName: string;
  avatarUrl?: string;
  email: string;
  phone?: string;
  location?: string;
  currentTitle?: string;
  source: CandidateSource;          // где нашли: LinkedIn, Referral, Website, Agency...
  status: CandidateStatus;          // общий статус, не путать со стадией в конкретной вакансии
  appliedJobIds: string[];          // на какие вакансии откликнулся / привязан
  resumeUrl?: string;
  tags: string[];
  createdAt: string;                // ISO date
  updatedAt: string;
}

type CandidateSource = 'linkedin' | 'referral' | 'website' | 'agency' | 'other';
type CandidateStatus = 'active' | 'hired' | 'rejected' | 'on_hold';
```

## Job (`entities/job`)

```ts
interface Job {
  id: string;
  title: string;
  department: string;
  location: string;
  employmentType: 'full_time' | 'part_time' | 'contract';
  status: 'open' | 'on_hold' | 'closed';
  hiringManagerId: string;          // ссылка на User
  openedAt: string;
  closedAt?: string;
  description: string;
  stageIds: string[];               // порядок стадий пайплайна для этой вакансии
}
```

## Stage (`entities/stage`)

Стадии воронки. Можно захардкодить дефолтный набор, но смоделировать как сущность, чтобы Kanban
(День 5) не был завязан на строковые константы по всему коду.

```ts
interface Stage {
  id: string;
  name: string;           // 'New', 'Screening', 'Interview', 'Offer', 'Hired', 'Rejected'
  order: number;
  isTerminal: boolean;    // true для Hired/Rejected — влияет на цвет/поведение в Kanban
}
```

## PipelineEntry (`entities/pipeline`)

Связка кандидат↔вакансия↔стадия — то, чем реально двигает Kanban при drag&drop.
Один кандидат может быть в пайплайне нескольких вакансий одновременно, поэтому это отдельная сущность,
а не поле в `Candidate`.

```ts
interface PipelineEntry {
  id: string;
  candidateId: string;
  jobId: string;
  stageId: string;
  movedAt: string;        // когда попал в текущую стадию — нужно для time-in-stage метрик
  rejectionReason?: string;
}
```

## Interview (`entities/interview`)

```ts
interface Interview {
  id: string;
  pipelineEntryId: string;
  scheduledAt: string;
  interviewerIds: string[];
  type: 'phone_screen' | 'technical' | 'onsite' | 'final';
  scorecard?: Scorecard;
}

interface Scorecard {
  overallRating: 1 | 2 | 3 | 4 | 5;
  criteria: { name: string; rating: 1 | 2 | 3 | 4 | 5; comment?: string }[];
  recommendation: 'strong_yes' | 'yes' | 'no' | 'strong_no';
}
```

## Note & Activity (`entities/note`)

Разделяем ручные заметки (Note) от системных событий (Activity), т.к. отображаются вместе в ленте
(День 6), но источники разные — так же, как в реальном ATS (Changes history vs заметки рекрутера).

```ts
interface Note {
  id: string;
  candidateId: string;
  authorId: string;
  text: string;
  createdAt: string;
}

// системное событие: смена стадии, назначено интервью, изменено поле и т.п.
interface ActivityEvent {
  id: string;
  candidateId: string;
  type: 'stage_changed' | 'interview_scheduled' | 'field_updated' | 'note_added';
  payload: Record<string, unknown>;  // например { from: 'Screening', to: 'Interview' }
  actorId: string;
  createdAt: string;
}
```

## User (`entities/user`)

Рекрутеры/hiring-менеджеры — минимально нужны для "Changed by" / "Hiring manager" полей.

```ts
interface User {
  id: string;
  fullName: string;
  avatarUrl?: string;
  role: 'recruiter' | 'hiring_manager' | 'admin';
}
```

## Связи между сущностями (кратко)

```
Job 1---N PipelineEntry N---1 Candidate
PipelineEntry 1---N Interview
Candidate 1---N Note
Candidate 1---N ActivityEvent
PipelineEntry N---1 Stage
Job N---1 User (hiringManagerId)
```

## Мок-данные — объём

Для Дня 2 (Dashboard) и Дня 3 (Candidates list) нужен достаточный объём, чтобы таблицы/графики не
выглядели пусто, но и не как в макете (556 сотрудников — избыточно для ручного тестирования):

- 40–60 Candidates
- 8–12 Jobs
- 6 Stages (New, Screening, Interview, Offer, Hired, Rejected)
- 60–100 PipelineEntry (кандидаты распределены по вакансиям и стадиям неравномерно — важно для
  реалистичного вида воронки на Dashboard)
- 15–20 Interview
- 30–50 Note + ActivityEvent вперемешку
- 5 User (2–3 recruiter, 2 hiring_manager)

Генерировать вручную не нужно — используйте библиотеку `@faker-js/faker` для полей (имена, email,
даты), но структуру и количество задавайте сами в `src/entities/*/mock/*.mock.ts`.
