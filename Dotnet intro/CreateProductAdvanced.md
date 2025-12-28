This tutorial is designed for a student working exclusively in a terminal using the **.NET CLI**. This hands-on approach builds a deep understanding of how code is organized and how the different layers of a professional .NET 8 application communicate.

---

### Step 1: Initialize the Solution and Project Layers

Open your terminal and execute these commands to build the structure of the **Nueva Americana** system.

```bash
# 1. Create and enter the root folder
mkdir NuevaAmericana && cd NuevaAmericana

# 2. Create the Solution
dotnet new sln -n NuevaAmericana

# 3. Create the 4 Architecture Layers
dotnet new classlib -n NuevaAmericana.Domain
dotnet new classlib -n NuevaAmericana.Infrastructure
dotnet new classlib -n NuevaAmericana.Services
dotnet new webapi -n NuevaAmericana.API

# 4. Connect Projects to the Solution
dotnet sln add **/*.csproj

```

After running the initial setup commands, verify the root structure.

**Terminal Command:**

```bash
tree /f  # Windows (PowerShell)
tree     # Linux/macOS

```

**Expected Output:**

```text
NuevaAmericana/
├── NuevaAmericana.sln
├── NuevaAmericana.API/
│   └── NuevaAmericana.API.csproj
├── NuevaAmericana.Domain/
│   └── NuevaAmericana.Domain.csproj
├── NuevaAmericana.Infrastructure/
│   └── NuevaAmericana.Infrastructure.csproj
└── NuevaAmericana.Services/
    └── NuevaAmericana.Services.csproj

```

---

### Step 2: Define Layer Dependencies

In Clean Architecture, dependencies flow inward. Run these commands to link the projects correctly:

```bash
# Infrastructure knows about Domain
dotnet add NuevaAmericana.Infrastructure/NuevaAmericana.Infrastructure.csproj reference NuevaAmericana.Domain/NuevaAmericana.Domain.csproj

# Services knows about Infrastructure (for Interfaces) and Domain
dotnet add NuevaAmericana.Services/NuevaAmericana.Services.csproj reference NuevaAmericana.Infrastructure/NuevaAmericana.Infrastructure.csproj
dotnet add NuevaAmericana.Services/NuevaAmericana.Services.csproj reference NuevaAmericana.Domain/NuevaAmericana.Domain.csproj

# API knows about Services
dotnet add NuevaAmericana.API/NuevaAmericana.API.csproj reference NuevaAmericana.Services/NuevaAmericana.Services.csproj

```

---

### Step 3: Scaffold the Directory Tree

Before writing code, create the folders that will hold our logic.

```bash
mkdir -p NuevaAmericana.Domain/Entities
mkdir -p NuevaAmericana.Infrastructure/Interfaces
mkdir -p NuevaAmericana.Infrastructure/Persistence/Repositories
mkdir -p NuevaAmericana.Services/Interfaces
mkdir -p NuevaAmericana.Services/DTOs
mkdir -p NuevaAmericana.Services/Mappings
mkdir -p NuevaAmericana.Services/Rules/ProductRules
mkdir -p NuevaAmericana.API/Controllers

```

After creating the internal directories, the student must ensure the "N-Tier" structure is ready for the code.

**Terminal Command:**

```bash
tree /f

```

**Expected Output:**

```text
NuevaAmericana/
├── NuevaAmericana.Domain/
│   └── Entities/
├── NuevaAmericana.Infrastructure/
│   ├── Interfaces/
│   └── Persistence/
│       └── Repositories/
├── NuevaAmericana.Services/
│   ├── DTOs/
│   ├── Interfaces/
│   ├── Mappings/
│   └── Rules/
│       └── ProductRules/
└── NuevaAmericana.API/
    └── Controllers/

```

---

### Step 4: Implementation (The Code Flow)

Students should use a terminal editor (like `nano`, `vim`, or `micro`) to create the following files. This flow follows a request from the bottom (**Domain**) to the top (**API**).

#### 4.1 The Domain Entity

**File:** `NuevaAmericana.Domain/Entities/Product.cs`

```csharp
namespace NuevaAmericana.Domain.Entities;
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Sku { get; set; } = string.Empty;
    public decimal Price { get; set; }
}

```

#### 4.2 The Repository Interface

**File:** `NuevaAmericana.Infrastructure/Interfaces/IProductRepository.cs`

```csharp
using NuevaAmericana.Domain.Entities;
namespace NuevaAmericana.Infrastructure.Interfaces;
public interface IProductRepository
{
    Task AddAsync(Product product);
    Task<bool> ExistsBySkuAsync(string sku);
}

```

#### 4.3 The Database Context (New)

**File:** `NuevaAmericana.Infrastructure/Persistence/AppDbContext.cs`

This class manages the connection to the database and maps our `Product` entity to a table.

```csharp
using Microsoft.EntityFrameworkCore;
using NuevaAmericana.Domain.Entities;

namespace NuevaAmericana.Infrastructure.Persistence;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>().HasIndex(p => p.Sku).IsUnique();
        base.OnModelCreating(modelBuilder);
    }
}

```

#### 4.4 The Repository Implementation (New)

**File:** `NuevaAmericana.Infrastructure/Persistence/Repositories/SqlProductRepository.cs`

This is the concrete class that uses the `AppDbContext` to perform actual database operations.

```csharp
using Microsoft.EntityFrameworkCore;
using NuevaAmericana.Domain.Entities;
using NuevaAmericana.Infrastructure.Interfaces;

namespace NuevaAmericana.Infrastructure.Persistence.Repositories;

public class SqlProductRepository : IProductRepository
{
    private readonly AppDbContext _context;

    public SqlProductRepository(AppDbContext context) => _context = context;

    public async Task AddAsync(Product product)
    {
        await _context.Products.AddAsync(product);
        await _context.SaveChangesAsync();
    }

    public async Task<bool> ExistsBySkuAsync(string sku)
    {
        return await _context.Products.AnyAsync(p => p.Sku == sku);
    }
}

```

#### 4.5 The Request DTO

**File:** `NuevaAmericana.Services/DTOs/ProductCreateRequest.cs`

```csharp
namespace NuevaAmericana.Services.DTOs;
public record ProductCreateRequest(string Name, string Sku, decimal Price);

```

#### 4.6 The Business Rule Interface

**File:** `NuevaAmericana.Services/Rules/IProductRule.cs`

```csharp
using NuevaAmericana.Services.DTOs;
namespace NuevaAmericana.Services.Rules;
public interface IProductRule
{
    string ErrorMessage { get; }
    Task<bool> IsValidAsync(ProductCreateRequest request);
}

```

#### 4.7 A Concrete Business Rule

**File:** `NuevaAmericana.Services/Rules/ProductRules/SkuUniqueRule.cs`

```csharp
using NuevaAmericana.Infrastructure.Interfaces;
using NuevaAmericana.Services.DTOs;
namespace NuevaAmericana.Services.Rules.ProductRules;
public class SkuUniqueRule : IProductRule
{
    private readonly IProductRepository _repo;
    public string ErrorMessage => "This SKU is already taken.";
    public SkuUniqueRule(IProductRepository repo) => _repo = repo;
    public async Task<bool> IsValidAsync(ProductCreateRequest request)
        => !await _repo.ExistsBySkuAsync(request.Sku);
}

```

#### 4.8 The Rules Engine (Orchestrator)

**File:** `NuevaAmericana.Services/Rules/ProductRulesEngine.cs`

```csharp
namespace NuevaAmericana.Services.Rules;
public class ProductRulesEngine
{
    private readonly IEnumerable<IProductRule> _rules;
    public ProductRulesEngine(IEnumerable<IProductRule> rules) => _rules = rules;
    public async Task<List<string>> RunAllAsync(ProductCreateRequest request)
    {
        var errors = new List<string>();
        foreach (var rule in _rules)
        {
            if (!await rule.IsValidAsync(request)) errors.Add(rule.ErrorMessage);
        }
        return errors;
    }
}

```

#### 4.9 The Mapping Extension

**File:** `NuevaAmericana.Services/Mappings/ProductMappingExtensions.cs`

```csharp
using NuevaAmericana.Domain.Entities;
using NuevaAmericana.Services.DTOs;
namespace NuevaAmericana.Services.Mappings;
public static class ProductMappingExtensions
{
    public static Product ToEntity(this ProductCreateRequest request) => new()
    {
        Name = request.Name,
        Sku = request.Sku,
        Price = request.Price
    };
}

```

#### 4.10 The Service Interface (New)

**File:** `NuevaAmericana.Services/Interfaces/IProductService.cs`

This interface defines the contract for the business logic layer, allowing the API to remain decoupled from the concrete implementation.

```csharp
using NuevaAmericana.Services.DTOs;

namespace NuevaAmericana.Services.Interfaces;

public interface IProductService
{
    Task CreateProductAsync(ProductCreateRequest request);
}

```

#### 4.11 The Service Layer Implementation

**File:** `NuevaAmericana.Services/ProductService.cs`

This class implements `IProductService` and orchestrates the business rules, mapping, and persistence.

```csharp
using NuevaAmericana.Infrastructure.Interfaces;
using NuevaAmericana.Services.DTOs;
using NuevaAmericana.Services.Rules;
using NuevaAmericana.Services.Mappings;
using NuevaAmericana.Services.Interfaces;

namespace NuevaAmericana.Services;

public class ProductService : IProductService
{
    private readonly IProductRepository _repo;
    private readonly ProductRulesEngine _rulesEngine;

    public ProductService(IProductRepository repo, ProductRulesEngine engine)
    {
        _repo = repo;
        _rulesEngine = engine;
    }

    public async Task CreateProductAsync(ProductCreateRequest request)
    {
        var errors = await _rulesEngine.RunAllAsync(request);

        if (errors.Any())
            throw new Exception(string.Join(", ", errors));

        var entity = request.ToEntity();
        await _repo.AddAsync(entity);
    }
}

```

#### 4.12 The Controller

**File:** `NuevaAmericana.API/Controllers/ProductsController.cs`

The controller now injects the `IProductService` interface.

```csharp
using Microsoft.AspNetCore.Mvc;
using NuevaAmericana.Services.Interfaces;
using NuevaAmericana.Services.DTOs;

namespace NuevaAmericana.API.Controllers;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _service;

    public ProductsController(IProductService service) => _service = service;

    [HttpPost]
    public async Task<IActionResult> Create(ProductCreateRequest request)
    {
        await _service.CreateProductAsync(request);
        return Ok("Product created successfully.");
    }
}

```

---

### Updated Final Structure

With these additions, the student's file tree should now look like this:

```text
NuevaAmericana/
├── NuevaAmericana.Domain/
│   └── Entities/
│       └── Product.cs
├── NuevaAmericana.Infrastructure/
│   ├── Interfaces/
│   │   └── IProductRepository.cs
│   └── Persistence/
│       ├── AppDbContext.cs
│       └── Repositories/
│           └── SqlProductRepository.cs
├── NuevaAmericana.Services/
│   ├── DTOs/
│   │   └── ProductCreateRequest.cs
│   ├── Interfaces/
│   │   └── IProductService.cs
│   ├── Mappings/
│   │   └── ProductMappingExtensions.cs
│   ├── Rules/
│   │   ├── IProductRule.cs
│   │   ├── ProductRulesEngine.cs
│   │   └── ProductRules/
│   │       └── SkuUniqueRule.cs
│   └── ProductService.cs
└── NuevaAmericana.API/
    └── Controllers/
        └── ProductsController.cs

```
---

To wire everything together, the student needs to configure the **Dependency Injection (DI)** container in the `Program.cs` file. This is where the application learns how to resolve interfaces (like `IProductService`) into concrete classes (like `ProductService`).

---

### Step 5: Configure Dependency Injection and Database

**File:** `NuevaAmericana.API/Program.cs`

Open the existing `Program.cs` in the API project and replace its contents with the following code. This configuration connects the layers and sets up an **In-Memory database** for quick testing.

```csharp
using Microsoft.EntityFrameworkCore;
using NuevaAmericana.Infrastructure.Interfaces;
using NuevaAmericana.Infrastructure.Persistence;
using NuevaAmericana.Infrastructure.Persistence.Repositories;
using NuevaAmericana.Services;
using NuevaAmericana.Services.Interfaces;
using NuevaAmericana.Services.Rules;
using NuevaAmericana.Services.Rules.ProductRules;

var builder = WebApplication.CreateBuilder(args);

// 1. Add Infrastructure: Database Context (Using In-Memory for testing)
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseInMemoryDatabase("NuevaAmericanaDb"));

// 2. Add Infrastructure: Repositories
builder.Services.AddScoped<IProductRepository, SqlProductRepository>();

// 3. Add Services: Business Rules
// Note: We register all IProductRule implementations so the Engine can inject them as a list
builder.Services.AddScoped<IProductRule, SkuUniqueRule>();
builder.Services.AddScoped<ProductRulesEngine>();

// 4. Add Services: Application Logic
builder.Services.AddScoped<IProductService, ProductService>();

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();

```

---

### Step 6: Install Required Packages

Because we are using **Entity Framework Core** and the **In-Memory provider**, the student must run these commands in the terminal to add the necessary libraries to the Infrastructure and API projects:

```bash
# Add EF Core to Infrastructure
dotnet add NuevaAmericana.Infrastructure/NuevaAmericana.Infrastructure.csproj package Microsoft.EntityFrameworkCore

# Add In-Memory Provider to API (for the Program.cs configuration)
dotnet add NuevaAmericana.API/NuevaAmericana.API.csproj package Microsoft.EntityFrameworkCore.InMemory

```

---

### Summary of the "Glue" Logic

| Registration Type | Description |
| --- | --- |
| **`AddDbContext`** | Enables the `AppDbContext` to be injected into repositories. |
| **`AddScoped`** | Creates a new instance of the service per HTTP request (standard for web APIs). |
| **IEnumerable Injection** | By registering multiple `IProductRule` types, the `ProductRulesEngine` automatically receives all of them in its constructor: `IEnumerable<IProductRule> rules`. |

**Would you like me to provide a `curl` command or a PowerShell script so the student can test the API from the terminal once it's running?**

### Step 7: Visual Map of Responsibilities (For the Student)

| Layer | File | Primary Responsibility |
| --- | --- | --- |
| **Domain** | `Product.cs` | Database Schema / Core State |
| **Infrastructure** | `IProductRepository.cs` | Contract for Persistence |
| **Services** | `ProductCreateRequest.cs` | API Contract (DTO) |
| **Services** | `IProductRule.cs` | Business Logic Contract |
| **Services** | `ProductService.cs` | Coordination & Workflow |
| **API** | `ProductsController.cs` | HTTP Communication |

---

### Step 8: Final Build and Run

Teach the student to verify their work via the CLI:

```bash
# Check for compilation errors
dotnet build

# Run the API
dotnet run --project NuevaAmericana.API

```

Would you like me to show you how to write the **Unit Test** for the `SkuUniqueRule` so the student can learn how to test these isolated rules using the terminal?
