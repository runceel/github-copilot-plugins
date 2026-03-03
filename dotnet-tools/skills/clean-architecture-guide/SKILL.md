---
name: clean-architecture-guide
description: Modular Monolith + Clean Architecture の設計ガイド。新しいモジュールの作成、画面の実装、機能追加を行う際には、必ずこのスキルを参照して従ってください。
---

# Clean Architecture 設計ガイド

このスキルは、本プロジェクトにおける Modular Monolith + Clean Architecture の具体的な設計パターンとルールを定義します。

> **重要:** このスキルに基づいてコード編集・生成を行う際には、対応する言語・プラットフォームのスキル（例: `dotnet10`、`aspnetcore-blazor10` 等）を必ず併せて参照してください。最新の言語機能や API を正しく使用するために必要です。

## 1. モジュール構成

新しいモジュールは以下の 3 プロジェクトで構成します。

```
src/Modules/<ModuleName>/
  <ModuleName>.Domain/           # エンティティ、値オブジェクト（依存なし）
  <ModuleName>.Application/      # UseCase、DTO、リポジトリインターフェース（Domain のみ依存）
  <ModuleName>.Infrastructure/   # リポジトリ実装、外部サービス連携、DI 登録（Application に依存）
```

### プロジェクト作成手順

```bash
# 1. プロジェクト作成
dotnet new classlib -n <ModuleName>.Domain -o src/Modules/<ModuleName>/<ModuleName>.Domain --framework net10.0
dotnet new classlib -n <ModuleName>.Application -o src/Modules/<ModuleName>/<ModuleName>.Application --framework net10.0
dotnet new classlib -n <ModuleName>.Infrastructure -o src/Modules/<ModuleName>/<ModuleName>.Infrastructure --framework net10.0

# 2. ソリューションに追加
dotnet sln add src/Modules/<ModuleName>/<ModuleName>.Domain
dotnet sln add src/Modules/<ModuleName>/<ModuleName>.Application
dotnet sln add src/Modules/<ModuleName>/<ModuleName>.Infrastructure

# 3. プロジェクト参照設定
dotnet add src/Modules/<ModuleName>/<ModuleName>.Application reference src/Modules/<ModuleName>/<ModuleName>.Domain
dotnet add src/Modules/<ModuleName>/<ModuleName>.Infrastructure reference src/Modules/<ModuleName>/<ModuleName>.Application

# 4. Infrastructure に DI パッケージ追加
dotnet add src/Modules/<ModuleName>/<ModuleName>.Infrastructure package Microsoft.Extensions.DependencyInjection.Abstractions

# 5. Infrastructure に EF Core パッケージ追加
dotnet add src/Modules/<ModuleName>/<ModuleName>.Infrastructure package Microsoft.EntityFrameworkCore
# DB プロバイダーパッケージも追加（例: SqlServer, Sqlite, Npgsql 等）
# ⚠️ プロバイダーが不明な場合はユーザーに確認すること
dotnet add src/Modules/<ModuleName>/<ModuleName>.Infrastructure package Microsoft.EntityFrameworkCore.<Provider>

# 6. Web からの参照追加
dotnet add src/Web reference src/Modules/<ModuleName>/<ModuleName>.Application
dotnet add src/Web reference src/Modules/<ModuleName>/<ModuleName>.Infrastructure

# 7. 自動生成された Class1.cs を各プロジェクトから削除
```

---

## 2. 依存関係ルール

```
Web ──→ Application (UseCase インターフェース、DTO のみ)
Web ──→ Infrastructure (DI 登録の拡張メソッド呼び出しのためのみ)

Infrastructure ──→ Application (リポジトリインターフェースの実装)
Application    ──→ Domain      (エンティティの利用)
Domain         ──→ (依存なし)
```

### 禁止事項

| ルール | 説明 |
|--------|------|
| **Web から Domain エンティティを直接参照しない** | Web 層は Application 層の DTO のみを使用する |
| **Web から Repository を直接注入しない** | Web 層は UseCase インターフェース経由でのみデータにアクセスする |
| **ビジネスロジックを Web / Infrastructure に書かない** | ビジネスロジックは Domain と Application のみに配置する |
| **Application 層から Infrastructure を参照しない** | Application 層はインターフェースを定義し、実装は Infrastructure 層が行う |
| **リポジトリ内で SaveChangesAsync を呼ばない** | SaveChangesAsync は UseCase が IUnitOfWork 経由で呼ぶ。1 UseCase = 1 トランザクション単位とする |

---

## 3. 各層の実装パターン

### 3.1 Domain 層 — エンティティ

ビジネスの中核となるデータ構造を定義します。

```csharp
namespace <ModuleName>.Domain;

public class <EntityName>
{
    public int ID { get; set; }
    public string Name { get; set; } = "";
    // ビジネスルールに関わるメソッドもここに配置可能
}
```

### 3.2 Application 層 — DTO

Web 層に公開するデータ転送オブジェクトです。Domain エンティティの代わりにこれを使います。

```csharp
namespace <ModuleName>.Application;

public record <EntityName>Dto(int ID, string Name);
```

### 3.3 Application 層 — リポジトリインターフェース

データアクセスの抽象を定義します。UseCase 内部から使用され、Web 層には公開しません。

```csharp
namespace <ModuleName>.Application;

public interface I<EntityName>Repository
{
    Task<IReadOnlyList<Domain.<EntityName>>> GetAllAsync();
    Task UpdateAsync(int id, ...);
}
```

### 3.4 Application 層 — IUnitOfWork インターフェース

Unit of Work の抽象を定義します。UseCase がトランザクションの完了（`SaveChangesAsync`）を制御するために使用します。

```csharp
namespace <ModuleName>.Application;

public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

> **ポイント:** `IUnitOfWork` を Application 層に置くことで、UseCase は Infrastructure 層（DbContext）に直接依存せずにトランザクション管理を行えます。

### 3.5 Application 層 — UseCase

**Web 層が唯一依存する操作インターフェース** です。1 つの操作 = 1 つの UseCase とし、`ExecuteAsync` メソッドを持ちます。

```csharp
namespace <ModuleName>.Application.UseCases;

// インターフェース
public interface IGet<EntityName>sUseCase
{
    Task<IReadOnlyList<<EntityName>Dto>> ExecuteAsync(string? searchText = null);
}

// 実装（Application 層内に配置）
public class Get<EntityName>sUseCase(I<EntityName>Repository repository) : IGet<EntityName>sUseCase
{
    public async Task<IReadOnlyList<<EntityName>Dto>> ExecuteAsync(string? searchText = null)
    {
        var entities = await repository.GetAllAsync();

        // フィルタリングなどのビジネスロジックはここで行う
        var filtered = string.IsNullOrEmpty(searchText)
            ? entities
            : entities.Where(e => e.Name.Contains(searchText, StringComparison.OrdinalIgnoreCase)).ToList();

        // Domain → DTO 変換
        return filtered.Select(e => new <EntityName>Dto(e.ID, e.Name)).ToList();
    }
}
```

```csharp
namespace <ModuleName>.Application.UseCases;

public interface IUpdate<EntityName>UseCase
{
    Task ExecuteAsync(int id, ...);
}

public class Update<EntityName>UseCase(I<EntityName>Repository repository, IUnitOfWork unitOfWork) : IUpdate<EntityName>UseCase
{
    public async Task ExecuteAsync(int id, ...)
    {
        await repository.UpdateAsync(id, ...);
        // Unit of Work: UseCase の最後で変更を永続化
        await unitOfWork.SaveChangesAsync();
    }
}
```

### 3.6 Infrastructure 層 — DbContext（Unit of Work 実装）

データアクセスには **Entity Framework Core** を使用します。`DbContext` が Application 層の `IUnitOfWork` を実装し、**Unit of Work** パターンの役割を担います。トランザクション管理と変更追跡を一括して行います。

```csharp
using <ModuleName>.Application;
using <ModuleName>.Domain;
using Microsoft.EntityFrameworkCore;

namespace <ModuleName>.Infrastructure;

public class <ModuleName>DbContext(DbContextOptions<<ModuleName>DbContext> options) : DbContext(options), IUnitOfWork
{
    public DbSet<<EntityName>> <EntityName>s => Set<<EntityName>>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // エンティティの設定（テーブル名、キー、制約など）
        modelBuilder.Entity<<EntityName>>(entity =>
        {
            entity.HasKey(e => e.ID);
            // 必要に応じてプロパティの設定を追加
        });
    }

    // SaveChangesAsync は DbContext に既に定義されているため、
    // IUnitOfWork.SaveChangesAsync として自動的に公開される
}
```

### 3.7 Infrastructure 層 — リポジトリ実装

リポジトリは `DbContext` を注入し、EF Core を使ってデータアクセスを行います。`SaveChangesAsync()` はリポジトリでは呼び出さず、UseCase 側で制御します。

```csharp
using <ModuleName>.Domain;
using Microsoft.EntityFrameworkCore;

namespace <ModuleName>.Infrastructure;

public class <EntityName>Repository(<ModuleName>DbContext dbContext) : Application.I<EntityName>Repository
{
    public async Task<IReadOnlyList<<EntityName>>> GetAllAsync()
    {
        return await dbContext.<EntityName>s.ToListAsync();
    }

    public async Task UpdateAsync(int id, ...)
    {
        var item = await dbContext.<EntityName>s.FindAsync(id);
        if (item is not null)
        {
            // プロパティの更新（変更追跡により自動で検出される）
        }
        // SaveChangesAsync は UseCase 側で呼ぶ
    }
}
```

### 3.8 Infrastructure 層 — DI 登録

モジュール単位で `Add<ModuleName>Module()` 拡張メソッドを提供します。

```csharp
using <ModuleName>.Application;
using <ModuleName>.Application.UseCases;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;

namespace <ModuleName>.Infrastructure;

public static class <ModuleName>ServiceCollectionExtensions
{
    public static IServiceCollection Add<ModuleName>Module(this IServiceCollection services, string connectionString)
    {
        // EF Core DbContext（Unit of Work）
        services.AddDbContext<<ModuleName>DbContext>(options =>
            options.UseSqlServer(connectionString));  // プロバイダーはプロジェクトに応じて変更
        services.AddScoped<IUnitOfWork>(sp => sp.GetRequiredService<<ModuleName>DbContext>());

        // Infrastructure（リポジトリ実装）
        services.AddScoped<I<EntityName>Repository, <EntityName>Repository>();

        // Application（UseCase）
        services.AddScoped<IGet<EntityName>sUseCase, Get<EntityName>sUseCase>();
        services.AddScoped<IUpdate<EntityName>UseCase, Update<EntityName>UseCase>();

        return services;
    }
}
```

---

## 4. Web 層（Blazor ページ）の実装パターン

Web 層は **UseCase インターフェースと DTO のみ** を使用します。

```razor
@page "/<route>"
@using <ModuleName>.Application
@using <ModuleName>.Application.UseCases
@rendermode InteractiveServer

<PageTitle>ページタイトル</PageTitle>

<h1>ページタイトル</h1>

@* UI コンポーネント *@

@code {
    [Inject]
    private IGet<EntityName>sUseCase Get<EntityName>sUseCase { get; set; } = default!;

    [Inject]
    private IUpdate<EntityName>UseCase Update<EntityName>UseCase { get; set; } = default!;

    private IReadOnlyList<<EntityName>Dto> items = [];

    protected override async Task OnInitializedAsync()
    {
        items = await Get<EntityName>sUseCase.ExecuteAsync();
    }
}
```

### Program.cs での登録

```csharp
using <ModuleName>.Infrastructure;

builder.Services.Add<ModuleName>Module(builder.Configuration.GetConnectionString("<ModuleName>")!);
```

---

## 5. チェックリスト

新しいモジュールや機能を実装する際に確認してください。

- [ ] Domain 層にエンティティを定義したか
- [ ] Application 層に DTO を定義したか
- [ ] Application 層にリポジトリインターフェースを定義したか
- [ ] Application 層に IUnitOfWork インターフェースを定義したか
- [ ] Application 層に UseCase（インターフェース + 実装）を定義したか
- [ ] Infrastructure 層に DbContext（Unit of Work）を定義したか
- [ ] Infrastructure 層にリポジトリ実装を配置したか（DbContext を注入し、SaveChangesAsync はリポジトリで呼ばない）
- [ ] UseCase で SaveChangesAsync を呼んで変更を永続化しているか
- [ ] Infrastructure 層に DI 登録用の拡張メソッドを作成したか
- [ ] Web 層は UseCase と DTO のみを参照しているか（Domain エンティティを直接参照していないか）
- [ ] Program.cs に `Add<ModuleName>Module()` を追加したか
- [ ] `dotnet build` が成功するか
