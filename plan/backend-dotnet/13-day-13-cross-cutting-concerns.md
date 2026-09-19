# День 13 — Валидация, обработка ошибок, логирование, документация

## Цель дня

Все эндпоинты из Дней 10–12 доведены до единого стандарта: валидация входных данных, единообразные
ошибки, структурированное логирование, полная Swagger-документация с примерами. Это день "инженерной
гигиены", а не новых фич.

## Пререквизиты

- Дни 8–12 завершены, все эндпоинты функциональны

## Файлы и папки к созданию

```
src/
  Ats.Application/
    Candidates/Validators/CreateNoteRequestValidator.cs
    Pipeline/Validators/MoveStageRequestValidator.cs
    Common/
      Exceptions/
        NotFoundException.cs
        ValidationFailedException.cs
      ProblemDetailsFactory.cs             (опционально, если стандартного недостаточно)
  Ats.Api/
    Middleware/
      ExceptionHandlingMiddleware.cs
    Program.cs                             (обновить: подключить middleware, FluentValidation, логирование)
```

## Пошаговые задачи

1. **FluentValidation**
   - Установить `FluentValidation` (+ `FluentValidation.DependencyInjectionExtensions`) в
     `Ats.Application`
   - `CreateNoteRequestValidator`: `Text` не пустой, максимальная длина (например 5000 символов),
     `AuthorId` не `Guid.Empty`
   - `MoveStageRequestValidator`: `StageId` не `Guid.Empty`
   - Зарегистрировать валидаторы в DI (`AddValidatorsFromAssembly`), вызывать явно в начале каждого
     эндпоинта, который принимает body (`var validationResult = await validator.ValidateAsync(request);
     if (!validationResult.IsValid) return Results.ValidationProblem(...)`) — для Minimal API это проще
     сделать явным вызовом, чем городить кастомный filter pipeline ради двух валидаторов на проекте

2. **Единообразная обработка ошибок**
   - `Common/Exceptions/NotFoundException.cs` — простое исключение с `EntityName`/`Id`
   - `Middleware/ExceptionHandlingMiddleware.cs` — ловит необработанные исключения, конвертирует в
     `ProblemDetails` (RFC 7807 — стандарт .NET для ошибок API): `NotFoundException` → 404,
     невалидные данные → 400, всё остальное → 500 с логированием полного stack trace (но без утечки
     деталей в тело ответа клиенту — только `traceId`)
   - Пройтись по сервисам с Дней 10–12: заменить точечные `return null`/`Results.NotFound()` внутри
     `Service`-методов на выброс `NotFoundException`, а HTTP-код формировать централизованно в
     middleware — сейчас, скорее всего, в паре мест это сделано непоследовательно (где-то возвращали
     `null` и проверяли в endpoint, где-то сразу `Results.NotFound()` внутри сервиса, что смешивает
     слои) — привести к одному стилю

3. **Логирование**
   - `Program.cs`: настроить `builder.Logging` (встроенный `ILogger` вполне достаточен для
     проекта, Serilog — опционально, если хочется потренировать отдельно)
   - Добавить логирование в ключевых местах: `MoveStageAsync` (кто, когда, из какой стадии в какую),
     необработанные исключения в middleware — с уровнем `Warning`/`Error` соответственно
   - Request-логирование по умолчанию в ASP.NET Core (`UseHttpLogging()` или встроенный
     `Microsoft.AspNetCore.HttpLogging`) — включить хотя бы в Development

4. **Swagger — довести до полноты**
   - Проверить, что каждый эндпоинт имеет: осмысленное summary (`.WithSummary("...")`), корректные
     типы ответов (`.Produces<T>(200)`, `.Produces(404)`, `.ProducesValidationProblem()`)
   - Сгруппировать эндпоинты по тегам (`.WithTags("Candidates")` и т.д.) — на 6+ групп эндпоинтов это
     реально влияет на читаемость Swagger UI

5. **Ретроспективный проход**
   - Пройтись по всем эндпоинтам из таблицы в `00-overview.md` и проверить на каждом: обрабатывается ли
     "не найдено", обрабатывается ли невалидный `Guid` в маршруте, есть ли смысловой `summary` в Swagger

6. **OpenAPI → TS-типы для фронтенда (contract-first)**
   - Теперь, когда Swagger полный и стабильный — самое время зафиксировать его как единый источник
     истины для типов на границе API, вместо того чтобы вручную поддерживать соответствие между
     C#-DTO и ручными TS-интерфейсами в `entities/*/model/types.ts`
   - На фронтенде: `npm install -D openapi-typescript`
   - Добавить npm-скрипт `"generate:api-types": "openapi-typescript http://localhost:5000/swagger/v1/swagger.json -o src/shared/api/generated/schema.d.ts"`
     (бэкенд должен быть запущен локально в момент генерации)
   - **Сознательно генерируем только типы, не хуки/клиент** (в отличие от orval, который умеет сразу
     сгенерировать TanStack Query hooks) — цель недели включает тренировку написания хуков руками,
     кодогенерация хуков это обесценивает. Типы — другое дело: они должны быть источником правды из
     контракта, а не дублироваться руками
   - Заменить импорты в `useCandidatesList`/`useJob`/`useJobPipelineEntries` и т.д.: вместо
     `CandidateListItemDto` из `entities/candidate/model/types.ts` — `components['schemas']['CandidateListItemDto']`
     из `shared/api/generated/schema.d.ts` (можно завести короткие type-алиасы в каждой feature,
     например `type CandidateListItemDto = components['schemas']['CandidateListItemDto']`, чтобы не
     писать длинный путь по всему коду)
   - Domain-модели в `entities/*/model/types.ts`, не являющиеся прямым слепком API-ответа (например,
     если появятся клиентские вычисляемые поля), оставить как есть — генерируются только типы,
     реально приходящие с бэкенда
   - Зафиксировать привычку: после любого изменения контракта на бэкенде (новое поле, новый эндпоинт) —
     перезапустить `npm run generate:api-types` **до** правки фронтенд-кода, а не после — тогда
     TypeScript сразу подсветит все места, которые нужно обновить под новый контракт

## Критерии готовности

- [ ] Запрос несуществующего `id` на любом detail-эндпоинте возвращает 404 с телом в формате ProblemDetails
- [ ] `POST /api/candidates/{id}/notes` с пустым `text` возвращает 400 с понятным сообщением об ошибке
  (не голый 500)
- [ ] Необработанное исключение (можно временно бросить `throw new Exception("test")` в одном месте и
  проверить) не роняет процесс и не отдаёт клиенту stack trace, только `traceId`
- [ ] Swagger UI сгруппирован по тегам, у каждого эндпоинта читаемое описание
- [ ] В логах видно структурированные записи при смене стадии кандидата
- [ ] `npm run generate:api-types` генерирует `schema.d.ts` без ошибок, фронтенд-хуки списков используют
  сгенерированные типы вместо ручных дублирующих интерфейсов

## Не делать сегодня

- Аутентификацию/авторизацию — по-прежнему вне скоупа
- Rate limiting, кэширование ответов — не оправдано для проекта на этом этапе
