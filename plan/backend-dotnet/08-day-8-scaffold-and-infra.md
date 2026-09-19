# День 8 — Скаффолд решения, Docker, EF Core, базовый Program.cs

## Цель дня

Поднятое пустое ASP.NET Core приложение, подключённое к Postgres в Docker через EF Core, с одним
health-check эндпоинтом и настроенной сериализацией JSON под фронтенд-контракт.

## Пререквизиты

- Прочитан `00-overview.md` и `01-domain-and-contracts.md`
- Установлен .NET 8 SDK, Docker Desktop (или аналог)

## Файлы и папки к созданию

```
backend/
  Ats.sln
  docker-compose.yml
  .dockerignore
  .gitignore
  src/
    Ats.Api/
      Ats.Api.csproj
      Program.cs
      appsettings.json
      appsettings.Development.json
      Endpoints/
        HealthEndpoints.cs
    Ats.Application/
      Ats.Application.csproj
    Ats.Domain/
      Ats.Domain.csproj
    Ats.Infrastructure/
      Ats.Infrastructure.csproj
      Persistence/
        AtsDbContext.cs
```

## Пошаговые задачи

1. **Solution и проекты**
   - `dotnet new sln -n Ats`
   - `dotnet new webapi -n Ats.Api -o src/Ats.Api` (можно с `--use-minimal-apis`, в .NET 8 это уже
     дефолт для `webapi` шаблона) — удалить сгенерированный `WeatherForecast`-мусор сразу
   - `dotnet new classlib -n Ats.Application -o src/Ats.Application`
   - `dotnet new classlib -n Ats.Domain -o src/Ats.Domain`
   - `dotnet new classlib -n Ats.Infrastructure -o src/Ats.Infrastructure`
   - `dotnet sln add src/**/*.csproj`
   - Настроить ссылки между проектами: `Ats.Api` → `Ats.Application` + `Ats.Infrastructure`;
     `Ats.Application` → `Ats.Domain`; `Ats.Infrastructure` → `Ats.Application` + `Ats.Domain`
     (`dotnet add <proj> reference <target>`)

2. **Docker Compose**
   - `docker-compose.yml`: один сервис `postgres` (образ `postgres:16`), с `environment` (`POSTGRES_DB`,
     `POSTGRES_USER`, `POSTGRES_PASSWORD`), `ports: 5432:5432`, `volumes` для персистентности между
     рестартами контейнера
   - Опционально: сервис `adminer` (образ `adminer`, порт 8080) — веб-UI для просмотра БД глазами без
     отдельного клиента типа DBeaver
   - `docker-compose up -d`, проверить `docker ps`

3. **EF Core + Npgsql**
   - В `Ats.Infrastructure`: `dotnet add package Microsoft.EntityFrameworkCore` +
     `Npgsql.EntityFrameworkCore.PostgreSQL` + `Microsoft.EntityFrameworkCore.Design`
   - `Persistence/AtsDbContext.cs` — пока пустой `DbContext` (наследник), без `DbSet`'ов — они появятся
     на Дне 9 вместе с сущностями. Задача сегодня — просто убедиться, что контекст создаётся и
     подключается к строке соединения

4. **Конфигурация и секреты**
   - **Не класть реальную строку подключения (с паролем) в `appsettings.Development.json`** — этот файл
     коммитится в git (не путать с `appsettings.*.local.json`/секретами), а пароль от локального Postgres
     всё равно рано или поздно окажется тем же паролем, который лень менять и на других окружениях —
     привычка "это же только локально" и есть типичная причина утечки. Вместо этого — .NET
     **user-secrets**: хранит значения вне репозитория, в `~/.microsoft/usersecrets/<id>/secrets.json`
     на вашей машине, подключается автоматически в Development-окружении
   - `dotnet user-secrets init --project src/Ats.Api` — добавляет `<UserSecretsId>` в `Ats.Api.csproj`
     (это просто GUID-метка, не секрет сам по себе — можно коммитить)
   - `dotnet user-secrets set "ConnectionStrings:Default" "Host=localhost;Port=5432;Database=ats;Username=ats;Password=<пароль-из-docker-compose>" --project src/Ats.Api`
   - В `appsettings.Development.json` оставить только несекретные настройки (уровни логирования и т.п.)
     либо секцию `ConnectionStrings` с **заведомо нерабочим плейсхолдером** (`"Default":
     "SET_VIA_USER_SECRETS"`) — так файл остаётся полезным как документация ожидаемой структуры
     конфигурации, не будучи источником реального пароля
   - В `Ats.Api/Program.cs`: `dotnet add package Microsoft.EntityFrameworkCore.Design` в `Ats.Api`
     тоже нужен для `dotnet ef` CLI; зарегистрировать `AddDbContext<AtsDbContext>(opts =>
     opts.UseNpgsql(builder.Configuration.GetConnectionString("Default")))` — `IConfiguration` сам
     сначала читает `appsettings.json` → `appsettings.Development.json` → user-secrets (Development) →
     переменные окружения, каждый следующий слой перекрывает предыдущий, поэтому плейсхолдер из
     `appsettings.Development.json` автоматически подменится реальным значением из user-secrets без
     дополнительного кода
   - На проде (День 14) секрет так же не хранится в файле — берётся из переменных окружения хостинга
     (`ConnectionStrings__Default`, двойное подчёркивание — стандартный для .NET способ задать
     вложенный ключ конфигурации через env var)

5. **JSON-сериализация под фронтенд**
   - В `Program.cs`: `builder.Services.ConfigureHttpJsonOptions(opts => { opts.SerializerOptions
     .PropertyNamingPolicy = JsonNamingPolicy.CamelCase; opts.SerializerOptions.Converters.Add(new
     JsonStringEnumConverter(JsonNamingPolicy.CamelCase)); })` — зафиксировать решение по enum'ам из
     `01-domain-and-contracts.md` именно здесь, одной точкой конфигурации на всё приложение

6. **CORS (заранее, пригодится с Дня 10)**
   - Настроить CORS-политику, разрешающую origin фронтенда (`http://localhost:5173` для Vite dev-сервера)
     — сделать сегодня, чтобы не спотыкаться об это посреди работы над эндпоинтами позже

7. **Health-check эндпоинт**
   - `Endpoints/HealthEndpoints.cs` — статический класс с extension-методом `MapHealthEndpoints(this
     WebApplication app)`, регистрирует `GET /health`, возвращающий `{ status: "ok", db: <true/false> }`
     (проверка БД — `dbContext.Database.CanConnectAsync()`)
   - Такой же паттерн (`Map<Feature>Endpoints`) переиспользуется для всех фич в следующие дни — не
     сваливать всё в `Program.cs`

8. **Swagger**
   - Подключить встроенный OpenAPI (`Microsoft.AspNetCore.OpenApi` + `Swashbuckle.AspNetCore` или
     новый `AddOpenApi()`/`MapOpenApi()` из .NET 8) — доступен на `/swagger` только в Development

## Критерии готовности

- [ ] `docker-compose up -d` поднимает Postgres, контейнер в статусе healthy
- [ ] `dotnet run --project src/Ats.Api` стартует без ошибок
- [ ] `GET /health` возвращает `{ "status": "ok", "db": true }`
- [ ] `/swagger` открывается в браузере
- [ ] Решение по enum-сериализации (camelCase vs PascalCase, snake_case) явно зафиксировано и записано
  (например, комментарием в `Program.cs` или отдельной строкой в этом файле плана)
- [ ] Реальная строка подключения с паролем нигде не закоммичена: `dotnet user-secrets init`/`set`
  выполнены, `appsettings.Development.json` содержит только плейсхолдер/несекретные настройки
- [ ] `git status` после `git add` не показывает файлов с реальными паролями (проверить глазами перед
  первым коммитом бэкенда — привычка из общего Git Safety Protocol, не разовая проверка только сегодня)

## Не делать сегодня

- Ни одной доменной сущности — только инфраструктура
- Аутентификация/авторизация — вне скоупа всей Фазы 2 целиком, если не останется сильно много
  свободного времени в конце
