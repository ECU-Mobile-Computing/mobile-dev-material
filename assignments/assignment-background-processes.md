# Assignment: MAUI Background Process

Build a .NET 11 MAUI app that writes raw sales transactions to **LiteDB**, aggregates them into **SQLite** with a periodic **background service**, and displays the aggregates on a two-tab **MVVM** UI (Dashboard + Sales).

## Assignment overview

In this assignment you will build **MauiBackgroundSalesDemo**, a mobile app that models a realistic on-device data pipeline:

1. The **Sales** tab writes raw sales transactions to **LiteDB**, a NoSQL document database that acts as the *system of record*.
2. A **background service** wakes up every 15 seconds, reads the raw transactions, and aggregates them by day, by product, and by category.
3. The aggregates are stored in **SQLite** relational summary tables - the *read model*.
4. The **Dashboard** tab (the default landing page) binds only to the SQLite summary tables, so its KPI cards and charts stay fast no matter how many raw transactions exist.

This raw-to-summary pattern is a small, on-device version of **ETL** (Extract → Transform → Load). Production systems use the same idea: the UI never scans full transaction history; it reads small, pre-computed summary tables that a background process keeps up to date.

After the app works, you will use the **MAUI DevFlow** tooling to list the app's files on the device, export both database files to your machine, and inspect them with external tools - proving you understand what each database actually contains and how the data flows between them.

## Learning objectives

By completing this assignment you will be able to:

1. Create and configure a .NET 11 MAUI project and add pinned NuGet package versions.
2. Model transactional data as LiteDB documents and summary data as relational SQLite tables, and explain when each storage style fits.
3. Implement a long-running worker with `BackgroundService` and `PeriodicTimer`, including cooperative cancellation with `CancellationToken`.
4. Apply the ETL pattern on-device: extract raw documents, transform them with LINQ `GroupBy`/`Sum` aggregations, and load relational summary tables.
5. Structure a MAUI app with MVVM using `CommunityToolkit.Mvvm` (`[ObservableProperty]`, `AsyncRelayCommand`, constructor injection).
6. Register and resolve services, view models, and pages with MAUI dependency injection, and explain the singleton lifetime choices.
7. Configure Shell `TabBar` navigation with a default landing tab.
8. Use MAUI DevFlow (broker, MCP server, agent, inspector, storage CLI) to browse an app's file system and export files from a device or emulator.
9. Inspect a LiteDB file and a SQLite file with external tools and verify that the aggregates mathematically match the raw transactions.

## Before you get started

### Prerequisites

Make sure your environment is ready before step 1:

- **.NET 11 SDK** - verify with `dotnet --version`
- **MAUI workload** - verify with `dotnet workload list` (install with `dotnet workload install maui` if missing)
- **A run target** - an Android emulator (or device), or Windows if you are on Windows
- **VS Code with the .NET MAUI extension** (or Visual Studio)
- **MAUI CLI** with DevFlow support - verify with `maui devflow --help`
- **SQLite client** tool options
  -- VS Code extension: alexcvzz.vscode-sqlite  
  -- DB Browser for SQLite (free) or equivalent, for inspecting the exported SQLite file
- **LiteDB client** tool options
  -- VS Code extension: jaufrdevosse.litedb-vscode
  -- LiteDB.Studio (free, Windows)
  -- Use a small console script, for inspecting the exported LiteDB file (scripts are provided if you want to try them)

### Key ideas to understand first

| | LiteDB (`sales-transactions.db`) | SQLite (`sales-aggregates.db`) |
|---|---|---|
| **Style** | NoSQL document database | Relational (tables, SQL) |
| **Role here** | System of record - every raw sale event | Read model - pre-computed summaries |
| **Written by** | Sales page (`LiteDbSalesStore`) | Background service + Sales page refresh |
| **Read by** | Background service (full scan each tick) | Dashboard page (`DashboardViewModel`) |
| **Shape** | `ProductSaleTransaction` documents in a `sales` collection | `DailySalesSummary`, `ProductSalesSummary`, `CategorySalesSummary` tables |

Two things that surprise most students - read these now so the later steps make sense:

- **MAUI does not auto-start hosted services.** In ASP.NET Core, `AddHostedService` is enough. In MAUI you must resolve the `BackgroundService` yourself and call `StartAsync` after `builder.Build()` (step 15 does this).
- **The databases are rebuilt, not patched.** Each aggregation tick deletes the summary rows and reinserts them from a full scan of LiteDB. That is simple and correct for a demo, and one of the reflection questions asks you when it stops being a good idea.

### Practical tips

- **Estimated time:** 3–5 hours including the DevFlow export and inspection work.
- **Commit after every numbered step** so you can roll back a bad paste.
- **Read each file before pasting it.** The reflection questions at the end ask about specific design decisions in this code.
- **Watch the debug log.** The line `Updated aggregate SQLite database with N sales transactions.` every 15 seconds is your proof that the background service is alive.
- **Build one platform first** (whichever you can run fastest), get it working, then worry about other targets.

Every step below is copy/paste ready. Follow them in order.

---

## 1. Create the project

From a terminal:

```bash
dotnet new maui -n MauiBackgroundSalesDemo -f net11.0
cd MauiBackgroundSalesDemo
```

## 2. Add the NuGet packages

```bash
dotnet add package LiteDB --version 5.0.21
dotnet add package Microsoft.Data.Sqlite --version 10.0.14
dotnet add package CommunityToolkit.Mvvm --version 8.4.2
```

Why these packages:

- **LiteDB** - raw device-side transactional sales records (system of record)
- **Microsoft.Data.Sqlite** - aggregate summary tables that the dashboard charts read
- **CommunityToolkit.Mvvm** - observable properties and commands for a clean MVVM shape

## 3. Create the folder structure

```bash
mkdir Models
mkdir Services
mkdir ViewModels
mkdir Views
```

## 4. Model the raw transactional data

Create `Models/ProductSaleTransaction.cs`:

```csharp
namespace MauiBackgroundSalesDemo.Models;

public class ProductSaleTransaction
{
    public int Id { get; set; }
    public DateTime Timestamp { get; set; } = DateTime.Now;
    public string Product { get; set; } = string.Empty;
    public int Quantity { get; set; } = 1;
    public decimal Price { get; set; } = 0m;
    public decimal Total => Quantity * Price;
}
```

## 5. Model the aggregate summary rows

Create `Models/SalesSummaryModels.cs`:

```csharp
namespace MauiBackgroundSalesDemo.Models;

public class DailySalesSummaryItem
{
    public string DateKey { get; set; } = string.Empty;
    public decimal TotalRevenue { get; set; }
    public int TransactionCount { get; set; }
}

public class ProductSalesSummaryItem
{
    public string Product { get; set; } = string.Empty;
    public decimal TotalRevenue { get; set; }
    public int UnitsSold { get; set; }
}

public class CategorySalesSummaryItem
{
    public string Category { get; set; } = string.Empty;
    public decimal TotalRevenue { get; set; }
    public int UnitsSold { get; set; }
}
```

## 6. Create the raw LiteDB store

Create `Services/LiteDbSalesStore.cs`. This service opens the LiteDB file in the app data folder, stores documents in a `sales` collection, inserts and reads transactions, and seeds demo data when the database is empty:

```csharp
using LiteDB;
using MauiBackgroundSalesDemo.Models;

namespace MauiBackgroundSalesDemo.Services;

public sealed class LiteDbSalesStore
{
    private readonly string _databasePath;

    public LiteDbSalesStore(string databasePath)
    {
        _databasePath = databasePath;
        var folder = Path.GetDirectoryName(databasePath);
        if (!string.IsNullOrWhiteSpace(folder))
        {
            Directory.CreateDirectory(folder);
        }
    }

    public async Task InitializeAsync(CancellationToken cancellationToken = default)
    {
        await Task.Run(() =>
        {
            using var database = new LiteDatabase(_databasePath);
            var sales = database.GetCollection<ProductSaleTransaction>("sales");
            sales.EnsureIndex(x => x.Timestamp);
            sales.EnsureIndex(x => x.Product);
        }, cancellationToken);
    }

    public async Task<List<ProductSaleTransaction>> GetTransactionsAsync(CancellationToken cancellationToken = default)
    {
        return await Task.Run(() =>
        {
            using var database = new LiteDatabase(_databasePath);
            return database
                .GetCollection<ProductSaleTransaction>("sales")
                .FindAll()
                .OrderByDescending(x => x.Timestamp)
                .ToList();
        }, cancellationToken);
    }

    public async Task AddTransactionAsync(ProductSaleTransaction transaction, CancellationToken cancellationToken = default)
    {
        if (string.IsNullOrWhiteSpace(transaction.Product))
        {
            throw new InvalidOperationException("Transaction product cannot be empty.");
        }

        await Task.Run(() =>
        {
            using var database = new LiteDatabase(_databasePath);
            var sales = database.GetCollection<ProductSaleTransaction>("sales");
            sales.Insert(transaction);
        }, cancellationToken);
    }

    public async Task EnsureSeedDataAsync(CancellationToken cancellationToken = default)
    {
        var transactions = await GetTransactionsAsync(cancellationToken);
        if (transactions.Count > 0)
        {
            return;
        }

        var starterData = new[]
        {
            new ProductSaleTransaction { Timestamp = DateTime.Now.AddDays(-5), Product = "Laptop", Quantity = 2, Price = 999.99m },
            new ProductSaleTransaction { Timestamp = DateTime.Now.AddDays(-4), Product = "Phone", Quantity = 4, Price = 699.00m },
            new ProductSaleTransaction { Timestamp = DateTime.Now.AddDays(-3), Product = "Monitor", Quantity = 1, Price = 329.50m },
            new ProductSaleTransaction { Timestamp = DateTime.Now.AddDays(-2), Product = "Keyboard", Quantity = 3, Price = 79.99m },
            new ProductSaleTransaction { Timestamp = DateTime.Now.AddDays(-1), Product = "Headphones", Quantity = 5, Price = 129.00m },
            new ProductSaleTransaction { Timestamp = DateTime.Now, Product = "Laptop", Quantity = 1, Price = 1249.00m },
            new ProductSaleTransaction { Timestamp = DateTime.Now, Product = "Desk Chair", Quantity = 2, Price = 245.00m },
            new ProductSaleTransaction { Timestamp = DateTime.Now.AddHours(-6), Product = "Phone", Quantity = 3, Price = 719.00m }
        };

        foreach (var transaction in starterData)
        {
            await AddTransactionAsync(transaction, cancellationToken);
        }
    }
}
```

## 7. Create the SQLite aggregate store

Create `Services/SqliteSalesAggregateStore.cs`. This service creates the three summary tables (`DailySalesSummary`, `ProductSalesSummary`, `CategorySalesSummary`), rebuilds them from the raw transactions, and exposes read methods for the dashboard:

```csharp
using MauiBackgroundSalesDemo.Models;
using Microsoft.Data.Sqlite;

namespace MauiBackgroundSalesDemo.Services;

public sealed class SqliteSalesAggregateStore
{
    private readonly string _databasePath;

    public SqliteSalesAggregateStore(string databasePath)
    {
        _databasePath = databasePath;
        var folder = Path.GetDirectoryName(databasePath);
        if (!string.IsNullOrWhiteSpace(folder))
        {
            Directory.CreateDirectory(folder);
        }
    }

    public async Task InitializeAsync(CancellationToken cancellationToken = default)
    {
        await using var connection = await OpenConnectionAsync(cancellationToken);
        await using (var command = connection.CreateCommand())
        {
            command.CommandText = @"
                CREATE TABLE IF NOT EXISTS DailySalesSummary (
                    DateKey TEXT PRIMARY KEY,
                    TotalRevenue REAL NOT NULL,
                    TransactionCount INTEGER NOT NULL
                );

                CREATE TABLE IF NOT EXISTS ProductSalesSummary (
                    Product TEXT PRIMARY KEY,
                    TotalRevenue REAL NOT NULL,
                    UnitsSold INTEGER NOT NULL
                );

                CREATE TABLE IF NOT EXISTS CategorySalesSummary (
                    Category TEXT PRIMARY KEY,
                    TotalRevenue REAL NOT NULL,
                    UnitsSold INTEGER NOT NULL
                );";

            await command.ExecuteNonQueryAsync(cancellationToken);
        }
    }

    public async Task UpdateFromTransactionsAsync(IEnumerable<ProductSaleTransaction> transactions, CancellationToken cancellationToken = default)
    {
        var allTransactions = transactions.ToList();
        var daily = allTransactions
            .GroupBy(x => x.Timestamp.Date)
            .Select(g => new DailySalesSummaryItem
            {
                DateKey = g.Key.ToString("yyyy-MM-dd"),
                TotalRevenue = g.Sum(x => x.Total),
                TransactionCount = g.Count()
            })
            .OrderBy(x => x.DateKey)
            .ToList();

        var products = allTransactions
            .GroupBy(x => x.Product)
            .Select(g => new ProductSalesSummaryItem
            {
                Product = g.Key,
                TotalRevenue = g.Sum(x => x.Total),
                UnitsSold = g.Sum(x => x.Quantity)
            })
            .OrderByDescending(x => x.TotalRevenue)
            .ToList();

        var categories = allTransactions
            .GroupBy(x => GetCategory(x.Product))
            .Select(g => new CategorySalesSummaryItem
            {
                Category = g.Key,
                TotalRevenue = g.Sum(x => x.Total),
                UnitsSold = g.Sum(x => x.Quantity)
            })
            .OrderByDescending(x => x.TotalRevenue)
            .ToList();

        await using var connection = await OpenConnectionAsync(cancellationToken);

        await using (var clearDaily = connection.CreateCommand())
        {
            clearDaily.CommandText = "DELETE FROM DailySalesSummary;";
            await clearDaily.ExecuteNonQueryAsync(cancellationToken);
        }

        await using (var clearProducts = connection.CreateCommand())
        {
            clearProducts.CommandText = "DELETE FROM ProductSalesSummary;";
            await clearProducts.ExecuteNonQueryAsync(cancellationToken);
        }

        await using (var clearCategories = connection.CreateCommand())
        {
            clearCategories.CommandText = "DELETE FROM CategorySalesSummary;";
            await clearCategories.ExecuteNonQueryAsync(cancellationToken);
        }

        foreach (var item in daily)
        {
            await using var insertDaily = connection.CreateCommand();
            insertDaily.CommandText = @"
                INSERT INTO DailySalesSummary (DateKey, TotalRevenue, TransactionCount)
                VALUES (@DateKey, @TotalRevenue, @TransactionCount);";
            insertDaily.Parameters.AddWithValue("@DateKey", item.DateKey);
            insertDaily.Parameters.AddWithValue("@TotalRevenue", Convert.ToDouble(item.TotalRevenue));
            insertDaily.Parameters.AddWithValue("@TransactionCount", item.TransactionCount);
            await insertDaily.ExecuteNonQueryAsync(cancellationToken);
        }

        foreach (var item in products)
        {
            await using var insertProducts = connection.CreateCommand();
            insertProducts.CommandText = @"
                INSERT INTO ProductSalesSummary (Product, TotalRevenue, UnitsSold)
                VALUES (@Product, @TotalRevenue, @UnitsSold);";
            insertProducts.Parameters.AddWithValue("@Product", item.Product);
            insertProducts.Parameters.AddWithValue("@TotalRevenue", Convert.ToDouble(item.TotalRevenue));
            insertProducts.Parameters.AddWithValue("@UnitsSold", item.UnitsSold);
            await insertProducts.ExecuteNonQueryAsync(cancellationToken);
        }

        foreach (var item in categories)
        {
            await using var insertCategories = connection.CreateCommand();
            insertCategories.CommandText = @"
                INSERT INTO CategorySalesSummary (Category, TotalRevenue, UnitsSold)
                VALUES (@Category, @TotalRevenue, @UnitsSold);";
            insertCategories.Parameters.AddWithValue("@Category", item.Category);
            insertCategories.Parameters.AddWithValue("@TotalRevenue", Convert.ToDouble(item.TotalRevenue));
            insertCategories.Parameters.AddWithValue("@UnitsSold", item.UnitsSold);
            await insertCategories.ExecuteNonQueryAsync(cancellationToken);
        }
    }

    public async Task<List<DailySalesSummaryItem>> GetDailySalesAsync(CancellationToken cancellationToken = default)
    {
        var rows = new List<DailySalesSummaryItem>();
        await using var connection = await OpenConnectionAsync(cancellationToken);
        await using var command = connection.CreateCommand();
        command.CommandText = "SELECT DateKey, TotalRevenue, TransactionCount FROM DailySalesSummary ORDER BY DateKey ASC;";

        await using var reader = await command.ExecuteReaderAsync(cancellationToken);
        while (await reader.ReadAsync(cancellationToken))
        {
            rows.Add(new DailySalesSummaryItem
            {
                DateKey = reader.GetString(0),
                TotalRevenue = Convert.ToDecimal(reader.GetDouble(1)),
                TransactionCount = reader.GetInt32(2)
            });
        }

        return rows;
    }

    public async Task<List<ProductSalesSummaryItem>> GetProductSalesAsync(CancellationToken cancellationToken = default)
    {
        var rows = new List<ProductSalesSummaryItem>();
        await using var connection = await OpenConnectionAsync(cancellationToken);
        await using var command = connection.CreateCommand();
        command.CommandText = "SELECT Product, TotalRevenue, UnitsSold FROM ProductSalesSummary ORDER BY TotalRevenue DESC;";

        await using var reader = await command.ExecuteReaderAsync(cancellationToken);
        while (await reader.ReadAsync(cancellationToken))
        {
            rows.Add(new ProductSalesSummaryItem
            {
                Product = reader.GetString(0),
                TotalRevenue = Convert.ToDecimal(reader.GetDouble(1)),
                UnitsSold = reader.GetInt32(2)
            });
        }

        return rows;
    }

    public async Task<List<CategorySalesSummaryItem>> GetCategorySalesAsync(CancellationToken cancellationToken = default)
    {
        var rows = new List<CategorySalesSummaryItem>();
        await using var connection = await OpenConnectionAsync(cancellationToken);
        await using var command = connection.CreateCommand();
        command.CommandText = "SELECT Category, TotalRevenue, UnitsSold FROM CategorySalesSummary ORDER BY TotalRevenue DESC;";

        await using var reader = await command.ExecuteReaderAsync(cancellationToken);
        while (await reader.ReadAsync(cancellationToken))
        {
            rows.Add(new CategorySalesSummaryItem
            {
                Category = reader.GetString(0),
                TotalRevenue = Convert.ToDecimal(reader.GetDouble(1)),
                UnitsSold = reader.GetInt32(2)
            });
        }

        return rows;
    }

    private static string GetCategory(string product)
    {
        var normalized = product.Trim();

        if (normalized.Contains("Laptop", StringComparison.OrdinalIgnoreCase) ||
            normalized.Contains("Phone", StringComparison.OrdinalIgnoreCase) ||
            normalized.Contains("Tablet", StringComparison.OrdinalIgnoreCase) ||
            normalized.Contains("Monitor", StringComparison.OrdinalIgnoreCase) ||
            normalized.Contains("Keyboard", StringComparison.OrdinalIgnoreCase) ||
            normalized.Contains("Mouse", StringComparison.OrdinalIgnoreCase) ||
            normalized.Contains("Headphones", StringComparison.OrdinalIgnoreCase))
        {
            return "Electronics";
        }

        if (normalized.Contains("Desk", StringComparison.OrdinalIgnoreCase) ||
            normalized.Contains("Chair", StringComparison.OrdinalIgnoreCase) ||
            normalized.Contains("Lamp", StringComparison.OrdinalIgnoreCase) ||
            normalized.Contains("Shelf", StringComparison.OrdinalIgnoreCase))
        {
            return "Office Furniture";
        }

        if (normalized.Contains("Shirt", StringComparison.OrdinalIgnoreCase) ||
            normalized.Contains("Jacket", StringComparison.OrdinalIgnoreCase) ||
            normalized.Contains("Sneakers", StringComparison.OrdinalIgnoreCase) ||
            normalized.Contains("Hat", StringComparison.OrdinalIgnoreCase))
        {
            return "Apparel";
        }

        return "General";
    }

    private async Task<SqliteConnection> OpenConnectionAsync(CancellationToken cancellationToken)
    {
        var connection = new SqliteConnection($"Data Source={_databasePath}");
        await connection.OpenAsync(cancellationToken);
        return connection;
    }
}
```

## 8. Create the background ETL service

Create `Services/SalesAggregationBackgroundService.cs`. On startup it initializes both databases, seeds demo data, and runs an immediate aggregation; after that a `PeriodicTimer` re-aggregates every 15 seconds:

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

namespace MauiBackgroundSalesDemo.Services;

public sealed class SalesAggregationBackgroundService : BackgroundService
{
    private readonly LiteDbSalesStore _liteDbSalesStore;
    private readonly SqliteSalesAggregateStore _sqliteSalesAggregateStore;
    private readonly ILogger<SalesAggregationBackgroundService> _logger;

    public SalesAggregationBackgroundService(
        LiteDbSalesStore liteDbSalesStore,
        SqliteSalesAggregateStore sqliteSalesAggregateStore,
        ILogger<SalesAggregationBackgroundService> logger)
    {
        _liteDbSalesStore = liteDbSalesStore;
        _sqliteSalesAggregateStore = sqliteSalesAggregateStore;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        try
        {
            await _liteDbSalesStore.InitializeAsync(stoppingToken);
            await _sqliteSalesAggregateStore.InitializeAsync(stoppingToken);
            await _liteDbSalesStore.EnsureSeedDataAsync(stoppingToken);
            await RefreshAggregateAsync(stoppingToken);

            using var timer = new PeriodicTimer(TimeSpan.FromSeconds(15));
            while (await timer.WaitForNextTickAsync(stoppingToken))
            {
                await RefreshAggregateAsync(stoppingToken);
            }
        }
        catch (OperationCanceledException)
        {
            _logger.LogInformation("Sales aggregation background service stopped.");
        }
    }

    private async Task RefreshAggregateAsync(CancellationToken stoppingToken)
    {
        var transactions = await _liteDbSalesStore.GetTransactionsAsync(stoppingToken);

        if (transactions.Count == 0)
        {
            return;
        }

        await _sqliteSalesAggregateStore.UpdateFromTransactionsAsync(transactions, stoppingToken);
        _logger.LogInformation("Updated aggregate SQLite database with {Count} sales transactions.", transactions.Count);
    }
}
```

## 9. Create the sales data models

Create `Models/DailySalesChartItem.cs`.
```csharp
public class DailySalesChartItem
{
    public string DateLabel { get; set; } = string.Empty;
    public decimal Revenue { get; set; }
    public double RelativeVolume { get; set; }
    public string RevenueDisplay => Revenue.ToString("C");
}
```

Create `Models/ProductSalesChartItem.cs`.
```csharp

public class ProductSalesChartItem
{
    public string Product { get; set; } = string.Empty;
    public decimal Revenue { get; set; }
    public int UnitsSold { get; set; }
    public double RelativeVolume { get; set; }
    public string RevenueDisplay => Revenue.ToString("C");
}
```

Create `Models/CategorySalesChartItem.cs`.
```csharp
public class CategorySalesChartItem
{
    public string Category { get; set; } = string.Empty;
    public decimal Revenue { get; set; }
    public int UnitsSold { get; set; }
    public double RelativeVolume { get; set; }
    public string RevenueDisplay => Revenue.ToString("C");
}
```

## 10. Create the Dashboard view model

Create `ViewModels/DashboardViewModel.cs`. It reads the aggregate rows from SQLite, computes KPI totals, and converts each row into a chart item with a `RelativeVolume` (0–1) that drives the `ProgressBar` bars:

```csharp
using System.Collections.ObjectModel;
using CommunityToolkit.Mvvm.ComponentModel;
using MauiBackgroundSalesDemo.Services;

namespace MauiBackgroundSalesDemo.ViewModels;

public partial class DashboardViewModel : ObservableObject
{
    private readonly SqliteSalesAggregateStore _aggregateStore;

    public ObservableCollection<DailySalesChartItem> DailySales { get; } = new();
    public ObservableCollection<ProductSalesChartItem> ProductSales { get; } = new();
    public ObservableCollection<CategorySalesChartItem> CategorySales { get; } = new();

    [ObservableProperty]
    private decimal totalRevenue;

    [ObservableProperty]
    private int totalTransactions;

    [ObservableProperty]
    private decimal averageSale;

    [ObservableProperty]
    private string topProduct = "Waiting for data...";

    public DashboardViewModel(SqliteSalesAggregateStore aggregateStore)
    {
        _aggregateStore = aggregateStore;
    }

    public async Task RefreshAsync()
    {
        var dailySummary = await _aggregateStore.GetDailySalesAsync();
        var productSummary = await _aggregateStore.GetProductSalesAsync();
        var categorySummary = await _aggregateStore.GetCategorySalesAsync();

        DailySales.Clear();
        ProductSales.Clear();
        CategorySales.Clear();

        if (dailySummary.Count == 0)
        {
            TotalRevenue = 0m;
            TotalTransactions = 0;
            AverageSale = 0m;
            TopProduct = "No sales yet";
            return;
        }

        var maxDailyRevenue = dailySummary.Max(x => x.TotalRevenue);
        var maxProductRevenue = productSummary.Count > 0 ? productSummary.Max(x => x.TotalRevenue) : 1m;
        var maxCategoryRevenue = categorySummary.Count > 0 ? categorySummary.Max(x => x.TotalRevenue) : 1m;

        foreach (var item in dailySummary)
        {
            var displayedDate = DateTime.TryParse(item.DateKey, out var date)
                ? date.ToString("MMM d")
                : item.DateKey;

            DailySales.Add(new DailySalesChartItem
            {
                DateLabel = displayedDate,
                Revenue = item.TotalRevenue,
                RelativeVolume = maxDailyRevenue == 0 ? 0d : (double)(item.TotalRevenue / maxDailyRevenue)
            });
        }

        foreach (var item in productSummary)
        {
            ProductSales.Add(new ProductSalesChartItem
            {
                Product = item.Product,
                Revenue = item.TotalRevenue,
                UnitsSold = item.UnitsSold,
                RelativeVolume = maxProductRevenue == 0 ? 0d : (double)(item.TotalRevenue / maxProductRevenue)
            });
        }

        foreach (var item in categorySummary)
        {
            CategorySales.Add(new CategorySalesChartItem
            {
                Category = item.Category,
                Revenue = item.TotalRevenue,
                UnitsSold = item.UnitsSold,
                RelativeVolume = maxCategoryRevenue == 0 ? 0d : (double)(item.TotalRevenue / maxCategoryRevenue)
            });
        }

        TotalRevenue = dailySummary.Sum(x => x.TotalRevenue);
        TotalTransactions = dailySummary.Sum(x => x.TransactionCount);
        AverageSale = TotalTransactions == 0 ? 0m : TotalRevenue / TotalTransactions;
        TopProduct = productSummary.OrderByDescending(x => x.TotalRevenue).FirstOrDefault()?.Product ?? "No sales yet";
    }
}


```

## 11. Create the Sales view model

Create `ViewModels/SalesViewModel.cs`. It validates the input, inserts a `ProductSaleTransaction` into LiteDB, and immediately refreshes the SQLite aggregates so the dashboard updates without waiting for the next timer tick:

```csharp
using System.Collections.ObjectModel;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using MauiBackgroundSalesDemo.Models;
using MauiBackgroundSalesDemo.Services;

namespace MauiBackgroundSalesDemo.ViewModels;

public partial class SalesViewModel : ObservableObject
{
    private readonly LiteDbSalesStore _liteDbSalesStore;
    private readonly SqliteSalesAggregateStore _sqliteSalesAggregateStore;

    public ObservableCollection<string> AvailableProducts { get; } = new()
    {
        "Laptop",
        "Phone",
        "Tablet",
        "Monitor",
        "Keyboard",
        "Headphones",
        "Desk Chair"
    };

    [ObservableProperty]
    private string selectedProduct = "Laptop";

    [ObservableProperty]
    private int quantity = 1;

    [ObservableProperty]
    private decimal price = 999.99m;

    [ObservableProperty]
    private string statusMessage = "Ready to add a new sale.";

    public IAsyncRelayCommand AddTransactionCommand { get; }

    public SalesViewModel(LiteDbSalesStore liteDbSalesStore, SqliteSalesAggregateStore sqliteSalesAggregateStore)
    {
        _liteDbSalesStore = liteDbSalesStore;
        _sqliteSalesAggregateStore = sqliteSalesAggregateStore;
        AddTransactionCommand = new AsyncRelayCommand(AddTransactionAsync);
        Price = GetSuggestedPrice(SelectedProduct);
    }

    partial void OnSelectedProductChanged(string value)
    {
        Price = GetSuggestedPrice(value);
    }

    private async Task AddTransactionAsync()
    {
        if (string.IsNullOrWhiteSpace(SelectedProduct))
        {
            StatusMessage = "Choose a product before saving.";
            return;
        }

        if (Quantity <= 0)
        {
            StatusMessage = "Quantity must be greater than zero.";
            return;
        }

        if (Price <= 0)
        {
            StatusMessage = "Price must be greater than zero.";
            return;
        }

        var transaction = new ProductSaleTransaction
        {
            Timestamp = DateTime.Now,
            Product = SelectedProduct,
            Quantity = Quantity,
            Price = Price
        };

        await _liteDbSalesStore.AddTransactionAsync(transaction);
        var allTransactions = await _liteDbSalesStore.GetTransactionsAsync();
        await _sqliteSalesAggregateStore.UpdateFromTransactionsAsync(allTransactions);

        StatusMessage = $"Saved {transaction.Product} x {transaction.Quantity} for {transaction.Total:C}.";
        Quantity = 1;
        Price = GetSuggestedPrice(SelectedProduct);
    }

    private static decimal GetSuggestedPrice(string product)
    {
        return product switch
        {
            "Laptop" => 999.99m,
            "Phone" => 699.00m,
            "Tablet" => 449.50m,
            "Monitor" => 329.99m,
            "Keyboard" => 79.99m,
            "Headphones" => 129.00m,
            "Desk Chair" => 245.00m,
            _ => 49.99m
        };
    }
}
```

## 12. Create the Dashboard page

Create `Views/DashboardPage.xaml`:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage
    x:Class="MauiBackgroundSalesDemo.Views.DashboardPage"
    xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
    Title="Dashboard">

    <ScrollView>
        <VerticalStackLayout Padding="20" Spacing="12">
            <Label Text="Sales Dashboard" FontSize="28" FontAttributes="Bold" />

            <Grid ColumnDefinitions="2*,1.5*,2*,2*" ColumnSpacing="8" RowDefinitions="Auto">
                <Border Grid.Column="0" Padding="10" StrokeThickness="1" StrokeShape="RoundRectangle 12">
                    <VerticalStackLayout>
                        <Label Text="Revenue" FontSize="12" TextColor="Gray" />
                        <Label Text="{Binding TotalRevenue, StringFormat='{0:C}'}" FontSize="14" FontAttributes="Bold" />
                    </VerticalStackLayout>
                </Border>

                <Border Grid.Column="1" Padding="10" StrokeThickness="1" StrokeShape="RoundRectangle 12">
                    <VerticalStackLayout>
                        <Label Text="Trans" FontSize="12" TextColor="Gray" />
                        <Label Text="{Binding TotalTransactions}" FontSize="18" FontAttributes="Bold" />
                    </VerticalStackLayout>
                </Border>

                <Border Grid.Column="2" Padding="10" StrokeThickness="1" StrokeShape="RoundRectangle 12">
                    <VerticalStackLayout>
                        <Label Text="Avg Sale" FontSize="12" TextColor="Gray" />
                        <Label Text="{Binding AverageSale, StringFormat='{0:C}'}" FontSize="14" FontAttributes="Bold" />
                    </VerticalStackLayout>
                </Border>

                <Border Grid.Column="3" Padding="10" StrokeThickness="1" StrokeShape="RoundRectangle 12">
                    <VerticalStackLayout>
                        <Label Text="Top Prod" FontSize="12" TextColor="Gray" />
                        <Label Text="{Binding TopProduct}" FontSize="16" FontAttributes="Bold" LineBreakMode="TailTruncation" />
                    </VerticalStackLayout>
                </Border>
            </Grid>

            <Label Text="Daily revenue trend" FontSize="18" FontAttributes="Bold" />
            <CollectionView ItemsSource="{Binding DailySales}" HeightRequest="180">
                <CollectionView.ItemTemplate>
                    <DataTemplate>
                        <Grid ColumnDefinitions="56,*,70" Padding="0,4">
                            <Label Text="{Binding DateLabel}" FontSize="12" VerticalTextAlignment="Center" />
                            <ProgressBar Grid.Column="1" Progress="{Binding RelativeVolume}" ProgressColor="#5B7CFA" />
                            <Label Grid.Column="2" Text="{Binding RevenueDisplay}" FontSize="12" HorizontalTextAlignment="End" VerticalTextAlignment="Center" />
                        </Grid>
                    </DataTemplate>
                </CollectionView.ItemTemplate>
            </CollectionView>

            <Label Text="Top products" FontSize="18" FontAttributes="Bold" />
            <CollectionView ItemsSource="{Binding ProductSales}" HeightRequest="200">
                <CollectionView.ItemTemplate>
                    <DataTemplate>
                        <Grid ColumnDefinitions="95,*,70" Padding="0,4">
                            <Label Text="{Binding Product}" FontSize="12" LineBreakMode="TailTruncation" />
                            <ProgressBar Grid.Column="1" Progress="{Binding RelativeVolume}" ProgressColor="#2DB56C" />
                            <Label Grid.Column="2" Text="{Binding RevenueDisplay}" FontSize="12" HorizontalTextAlignment="End" />
                        </Grid>
                    </DataTemplate>
                </CollectionView.ItemTemplate>
            </CollectionView>

            <Label Text="Category totals" FontSize="18" FontAttributes="Bold" />
            <CollectionView ItemsSource="{Binding CategorySales}" HeightRequest="180">
                <CollectionView.ItemTemplate>
                    <DataTemplate>
                        <Grid ColumnDefinitions="110,*,70" Padding="0,4">
                            <Label Text="{Binding Category}" FontSize="12" LineBreakMode="TailTruncation" />
                            <ProgressBar Grid.Column="1" Progress="{Binding RelativeVolume}" ProgressColor="#F0A500" />
                            <Label Grid.Column="2" Text="{Binding RevenueDisplay}" FontSize="12" HorizontalTextAlignment="End" />
                        </Grid>
                    </DataTemplate>
                </CollectionView.ItemTemplate>
            </CollectionView>
        </VerticalStackLayout>
    </ScrollView>
</ContentPage>

```

Create `Views/DashboardPage.xaml.cs`. It refreshes the view model every time the page appears, so new aggregates show up when the user switches tabs:

```csharp
using MauiBackgroundSalesDemo.ViewModels;

namespace MauiBackgroundSalesDemo.Views;

public partial class DashboardPage : ContentPage
{
    public DashboardPage(DashboardViewModel viewModel)
    {
        InitializeComponent();
        BindingContext = viewModel;
    }

    protected override async void OnAppearing()
    {
        base.OnAppearing();
        if (BindingContext is DashboardViewModel viewModel)
        {
            await viewModel.RefreshAsync();
        }
    }
}
```

## 13. Create the Sales page

Create `Views/SalesPage.xaml`:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage
    x:Class="MauiBackgroundSalesDemo.Views.SalesPage"
    xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
    Title="Sales">

    <VerticalStackLayout Padding="20" Spacing="16">
        <Label Text="Add mock sales" FontSize="28" FontAttributes="Bold" />

        <Label Text="Product" FontSize="14" />
        <Picker ItemsSource="{Binding AvailableProducts}" SelectedItem="{Binding SelectedProduct}" />

        <Label Text="Quantity" FontSize="14" />
        <Entry Text="{Binding Quantity}" Keyboard="Numeric" />

        <Label Text="Price" FontSize="14" />
        <Entry Text="{Binding Price}" Keyboard="Numeric" />

        <Button
            Text="Save sale"
            Command="{Binding AddTransactionCommand}" />

        <Border Padding="12" StrokeThickness="1" StrokeShape="RoundRectangle 12">
            <Label Text="{Binding StatusMessage}" FontSize="14" />
        </Border>
    </VerticalStackLayout>
</ContentPage>
```

Create `Views/SalesPage.xaml.cs`:

```csharp
using MauiBackgroundSalesDemo.ViewModels;

namespace MauiBackgroundSalesDemo.Views;

public partial class SalesPage : ContentPage
{
    public SalesPage(SalesViewModel viewModel)
    {
        InitializeComponent();
        BindingContext = viewModel;
    }
}
```

## 14. Configure the two-tab shell

Replace the contents of `AppShell.xaml` with a `TabBar` that makes **Dashboard** the default landing tab:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<Shell
    x:Class="MauiBackgroundSalesDemo.AppShell"
    xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
    xmlns:views="clr-namespace:MauiBackgroundSalesDemo.Views"
    Shell.FlyoutBehavior="Disabled"
    Title="MauiBackgroundSalesDemo">

    <TabBar>
        <ShellContent
            Title="Dashboard"
            ContentTemplate="{DataTemplate views:DashboardPage}"
            Route="dashboard" />

        <ShellContent
            Title="Sales"
            ContentTemplate="{DataTemplate views:SalesPage}"
            Route="sales" />
    </TabBar>
</Shell>
```

`AppShell.xaml.cs` stays as the template default:

```csharp
namespace MauiBackgroundSalesDemo;

public partial class AppShell : Shell
{
    public AppShell()
    {
        InitializeComponent();
    }
}
```

The template's `MainPage.xaml` / `MainPage.xaml.cs` are no longer referenced; you can delete them or leave them in place.

## 15. Wire everything up in MauiProgram.cs

Replace the contents of `MauiProgram.cs`. This registers both stores (pointing at `FileSystem.AppDataDirectory`), the view models, the pages, and the background service - then starts the background service when the app is built:

```csharp
using MauiBackgroundSalesDemo.Services;
using MauiBackgroundSalesDemo.ViewModels;
using MauiBackgroundSalesDemo.Views;
using Microsoft.Extensions.Logging;

namespace MauiBackgroundSalesDemo;

public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        builder
            .UseMauiApp<App>()
            .ConfigureFonts(fonts =>
            {
                fonts.AddFont("OpenSans-Regular.ttf", "OpenSansRegular");
                fonts.AddFont("OpenSans-Semibold.ttf", "OpenSansSemibold");
            });

#if DEBUG
        builder.Logging.AddDebug();
#endif

        // Database stores (files live in the app data folder on every platform)
        builder.Services.AddSingleton(new LiteDbSalesStore(
            Path.Combine(FileSystem.AppDataDirectory, "sales-transactions.db")));
        builder.Services.AddSingleton(new SqliteSalesAggregateStore(
            Path.Combine(FileSystem.AppDataDirectory, "sales-aggregates.db")));

        // Background ETL service
        builder.Services.AddSingleton<SalesAggregationBackgroundService>();

        // View models
        builder.Services.AddSingleton<DashboardViewModel>();
        builder.Services.AddSingleton<SalesViewModel>();

        // Pages
        builder.Services.AddSingleton<DashboardPage>();
        builder.Services.AddSingleton<SalesPage>();

        var app = builder.Build();

        // MAUI does not auto-start IHostedService instances, so start it manually.
        var aggregationService = app.Services.GetRequiredService<SalesAggregationBackgroundService>();
        _ = aggregationService.StartAsync(CancellationToken.None);

        return app;
    }
}
```

> Note: `MauiApp` does not run hosted services the way a generic .NET host does, which is why the service is resolved and started explicitly after `builder.Build()`.

## 16. Build and run

Build for your platform(s):

```bash
dotnet build -f net11.0-android
```

Run on Windows:

```bash
dotnet run -f net11.0-windows10.0.19041.0
```

Run on Android (emulator or device):

```bash
dotnet run -f net11.0-android -c DEBUG
```

### What you should see

1. The app opens on the **Dashboard** tab with seeded demo data: KPI cards (Revenue, Transactions, Avg Sale, Top Product) plus three progress-bar charts (daily trend, top products, category totals).
2. Switch to the **Sales** tab, pick a product (the price auto-fills), set a quantity, and tap **Save sale**.
3. Switch back to **Dashboard** - the totals and charts reflect the new sale (the Sales page refreshes the aggregates immediately, and the background service also re-aggregates every 15 seconds).

## 17. Add DevFlow to the project

Add DevFlow package to your project
```bash
dotnet add package Microsoft.Maui.DevFlow.Agent --prerelease
```

In MauiProgram.cs, add the following using statement wrapped in a DEBUG conditional:
```csharp
#if DEBUG
    using Microsoft.Maui.DevFlow.Agent;
#endif
```

In MauiProgram.cs, add the following code right before the call to builder.Build().
```csharp
#if DEBUG
    builder.AddMauiDevFlowAgent();
#endif
```

Start the devflow broker
```bash
maui devflow broker start
```

Start the devflow MCP server
```bash
maui devflow mcp --platform android  # Specify the platform you are using e.g. windows, ios, android, maccatalyst
```

Wait for mobile application to connect to DevFlow.  Agent port (e.g. 9223, 10223, etc.) is displayed once agent connects to DevFlow.
```bash
maui devflow agent wait
```

Start the web application. Must start in DEBUG mode to enable DevFlow.
```bash
dotnet run -f net11.0-android -c DEBUG # e.g. 
```

Once agent is connected, you can open inspector in a browser  
```bash
http://localhost:19223/inspector
```

## 18. List the files

You can list your mobile app's files using inspector, but you need to use the maui CLI to download them.

List available files using the maui CLI.
```bash
maui devflow storage files list
# or
maui devflow storage files list "." --root appData
```

You should see the names of both the SQLite DB and the LiteDB in the output.
- sales-transactions.db (LiteDB)
- sales-aggregates.db (SQLite)
```json
{                 
  "root": "appData",
  "path": "",
  "entries": [
    {
      "name": ".__override__",
      "type": "directory",
      "lastModified": "2026-11-02T04:28:20.3979998Z"
    },
    {
      "name": ".local",
      "type": "directory",
      "lastModified": "2026-11-02T04:28:22.1619998Z"
    },
    {
      "name": ".config",
      "type": "directory",
      "lastModified": "2026-11-02T04:28:22.1619998Z"
    },
    {
      "name": "sales-transactions.db",
      "type": "file",
      "size": 49152,
      "lastModified": "2026-11-02T04:28:23.5379998Z"
    },
    {
      "name": "sales-aggregates.db",
      "type": "file",
      "size": 28672,
      "lastModified": "2026-11-02T12:33:04.6219183Z"
    },
    {
      "name": "profileInstalled",
      "type": "file",
      "size": 24,
      "lastModified": "2026-11-02T04:33:38.1299989Z"
    }
  ]
}
```


## 19. Export the databases with DevFlow

The app writes two files into `FileSystem.AppDataDirectory`:

- `sales-transactions.db` - LiteDB raw transactional database
- `sales-aggregates.db` - SQLite aggregated data used by the charts

Download files from emulator or device. The following commands will create a Data folder in your project if it does not already exist.
```bash
maui devflow storage files download "sales-transactions.db" --output ".\Data\sales-transactions.db"
maui devflow storage files download "sales-aggregates.db" --output ".\Data\sales-aggregates.db"
```


## 20. Inspect the exported databases

### LiteDB (raw transactions)

Use a LiteDB browser, or a small .NET script:

```csharp
using LiteDB;

using var db = new LiteDatabase("sales-transactions.db");
var sales = db.GetCollection<ProductSaleTransaction>("sales");
foreach (var item in sales.FindAll())
{
    Console.WriteLine($"{item.Timestamp} | {item.Product} | {item.Quantity} | {item.Price:C}");
}
```

### SQLite (aggregates)

Open with DB Browser for SQLite, or a small .NET script:

```csharp
using Microsoft.Data.Sqlite;

var connection = new SqliteConnection("Data Source=sales-aggregates.db");
await connection.OpenAsync();

var command = connection.CreateCommand();
command.CommandText = "SELECT * FROM DailySalesSummary;";

using var reader = await command.ExecuteReaderAsync();
while (await reader.ReadAsync())
{
    Console.WriteLine($"{reader[0]} | {reader[1]} | {reader[2]}");
}
```

Check these tables - their totals should reflect the raw transactions after the background aggregation runs:

- `DailySalesSummary`
- `ProductSalesSummary`
- `CategorySalesSummary`

## 21. Database inspection tasks

Complete every task below and capture the evidence listed. This is the proof that you understand what each database contains - it is graded.

### Task A - Document the files on the device

From the step 18 `maui devflow storage files list` output, record:

1. The name, size (bytes), and last-modified timestamp of `sales-transactions.db`.
2. The name, size (bytes), and last-modified timestamp of `sales-aggregates.db`.
3. The storage root they live under (`appData`, which maps to `FileSystem.AppDataDirectory`).

**Evidence:** paste the JSON listing output into your submission document.

### Task B - Inspect the LiteDB raw database

Open your exported `Data\sales-transactions.db` with LiteDB.Studio (or the console script from step 20) and:

1. Confirm the collection is named `sales`.
2. Record the **total document count**.
3. Capture the full listing of documents, showing `_id`, `Timestamp`, `Product`, `Quantity`, and `Price`.
4. Identify the 8 seeded demo records and at least 2 records you added from the Sales tab (match them by timestamp).
5. Note where the auto-assigned `_id` values come from - you never set `Id` in the app.

**Evidence:** screenshot of LiteDB.Studio (or console output) showing all documents, plus the document count.

If you use LiteDB.Studio, the query is:

```sql
SELECT $ FROM sales;
```

### Task C - Inspect the SQLite aggregate database

Open your exported `Data\sales-aggregates.db` with DB Browser for SQLite (or the console script from step 20) and run each query:

```sql
-- 1. Confirm the schema
SELECT name FROM sqlite_master WHERE type = 'table';

-- 2. Daily trend rows
SELECT * FROM DailySalesSummary ORDER BY DateKey;

-- 3. Product totals
SELECT * FROM ProductSalesSummary ORDER BY TotalRevenue DESC;

-- 4. Category totals
SELECT * FROM CategorySalesSummary ORDER BY TotalRevenue DESC;

-- 5. Cross-table check: both totals must be equal
SELECT (SELECT SUM(TotalRevenue) FROM DailySalesSummary)  AS DailyTotal,
       (SELECT SUM(TotalRevenue) FROM ProductSalesSummary) AS ProductTotal;
```

**Evidence:** screenshots (or pasted output) of all five query results.

### Task D - Prove the aggregation math

Pick **one product** (for example `Phone`) and verify the pipeline end to end:

1. In the LiteDB data from Task B, list every `sales` document for that product and compute `SUM(Quantity × Price)` and `SUM(Quantity)` by hand (a calculator or spreadsheet is fine - show your work).
2. In the SQLite data from Task C, find that product's row in `ProductSalesSummary`.
3. Confirm your hand-computed revenue matches `TotalRevenue` and your quantity sum matches `UnitsSold`.
4. State which category that product maps to (see `GetCategory` in `SqliteSalesAggregateStore.cs`) and confirm its revenue is included in that `CategorySalesSummary` row.

**Evidence:** your worked calculation next to the matching SQLite row.

### Task E - Prove the background service refreshes the aggregates

1. Note the current `TotalRevenue` for your chosen product in the exported SQLite file.
2. In the running app, add one more sale for that product from the Sales tab and record the quantity and price you used.
3. Wait at least 15 seconds (one background tick), then re-run the step 19 download commands to re-export `sales-aggregates.db`.
4. Re-run the Task C product query and show the row changed by exactly `Quantity × Price`.

**Evidence:** before and after query results, plus the sale you entered.

---

## Recap: final project layout

```text
MauiBackgroundSalesDemo/
├── MauiBackgroundSalesDemo.csproj   (LiteDB, Microsoft.Data.Sqlite, CommunityToolkit.Mvvm)
├── MauiProgram.cs                   (DI registration + background service startup)
├── AppShell.xaml / .cs              (TabBar: Dashboard + Sales)
├── Models/
│   ├── ProductSaleTransaction.cs    (raw LiteDB document)
│   └── SalesSummaryModels.cs        (daily / product / category summary rows)
├── Services/
│   ├── LiteDbSalesStore.cs          (raw transaction store + seed data)
│   ├── SqliteSalesAggregateStore.cs (summary tables + read/write)
│   └── SalesAggregationBackgroundService.cs (15-second ETL timer)
├── ViewModels/
│   ├── DashboardViewModel.cs        (KPIs + chart items from SQLite)
│   └── SalesViewModel.cs            (validation + insert into LiteDB)
└── Views/
    ├── DashboardPage.xaml / .cs     (KPI cards + 3 progress-bar charts)
    └── SalesPage.xaml / .cs         (picker, quantity, price, save button)
```

Data flow: **Sales page → LiteDB (`sales-transactions.db`) → background service (every 15 s) → SQLite (`sales-aggregates.db`) → Dashboard page**.

---

## Reflection questions

Answer all twelve questions in your own words. Short paragraphs (3–6 sentences) are expected; one-line answers will not earn full credit.

1. Why does this app use two databases instead of one? What would be lost if everything lived only in LiteDB? What would be lost if everything lived only in SQLite?
2. The LiteDB file is called the *system of record* and the SQLite file the *read model*. Explain both terms, and explain why the Dashboard should not query the raw transaction collection directly as the data grows.
3. Map the three letters of **ETL** onto specific methods in `SalesAggregationBackgroundService` and `SqliteSalesAggregateStore`. Which code extracts, which transforms, and which loads?
4. MAUI does not automatically start `IHostedService` instances the way an ASP.NET Core host does. How does this project start the background service, and what would the symptom be if that code were removed?
5. What does `PeriodicTimer` with `WaitForNextTickAsync` give you compared to a `while (true) { await Task.Delay(...); }` loop or a classic `System.Timers.Timer`?
6. Trace the `CancellationToken` from `ExecuteAsync` down into the store methods. What happens when the app shuts down, and why does the service catch `OperationCanceledException` instead of letting it escape?
7. Each aggregation tick deletes every summary row and reinserts them from a full scan of LiteDB. Why is that acceptable here, at what scale does it become a problem, and what incremental strategy (for example a timestamp watermark) could replace it?
8. Prices are `decimal` in C# but stored as `REAL` (floating point) in SQLite via `Convert.ToDouble`. What risk does that introduce for money values, and name one safer storage strategy.
9. The stores, view models, and pages are all registered as singletons. Explain why that works in this app, and describe one situation where a singleton page or view model would cause a bug.
10. `SalesViewModel` refreshes the SQLite aggregates immediately after saving a sale even though the background service would catch up within 15 seconds anyway. Why is the immediate refresh there, and what would the user experience be without it?
11. Why does `DashboardPage` call `RefreshAsync` in `OnAppearing` rather than only in the constructor? What user action would show stale data if `OnAppearing` were not overridden?
12. During DevFlow inspection you exported point-in-time copies of both files. Why might the two exported files briefly disagree with each other, and how did Task E account for that?

---

## Troubleshooting guide

Work top to bottom: confirm the project builds, then that the app runs, then that the background service ticks, then that DevFlow connects.

### Build and package problems

| Symptom | Likely cause and fix |
|---|---|
| `dotnet new maui` → "No templates found" | MAUI workload missing. Run `dotnet workload install maui`, then retry. |
| `NU1102` / version not found for a package | Typo in the version pin from step 2, or an offline feed. Re-run the exact commands in step 2 and check `dotnet nuget list source`. |
| `Microsoft.Maui.DevFlow.Agent` not found | It is prerelease - the install must include `--prerelease` (step 17). |
| `InitializeComponent` does not exist | The `x:Class` in a XAML file does not match the namespace/class in its `.xaml.cs`, or the file was created with the wrong name. Compare both against steps 12–13 exactly. |
| `XFC0000` / unknown XAML type `views:DashboardPage` | The `xmlns:views="clr-namespace:MauiBackgroundSalesDemo.Views"` declaration is missing or misspelled in `AppShell.xaml` (step 14). |
| Build errors mention `net11.0-windows` on macOS (or vice versa) | You are building a target your OS cannot build. Pass `-f` with a framework your machine supports (step 16). |
| Weird stale errors after lots of edits | Clean rebuild: `dotnet clean`, delete `bin` and `obj`, then `dotnet build` again. |

### Runtime problems

| Symptom | Likely cause and fix |
|---|---|
| App opens on the template "Hello, World" page | `AppShell.xaml` was not replaced with the `TabBar` version (step 14). |
| Crash on startup: "Unable to resolve service for type ..." | A registration in `MauiProgram.cs` is missing. Every page, view model, store, and the background service must be registered (step 15). |
| Dashboard shows "No sales yet" forever | The background service never started. Confirm the `aggregationService.StartAsync(...)` call after `builder.Build()` exists (step 15), and look for the log line `Updated aggregate SQLite database with N sales transactions.` every 15 s. |
| Dashboard KPIs stay at zero after adding a sale | You stayed on the Sales tab - the Dashboard refreshes in `OnAppearing`, so switch tabs. If it still fails, check for exceptions from `UpdateFromTransactionsAsync` in the debug output. |
| `SQLite Error 14: unable to open database file` | The directory does not exist or the path is wrong. Both store constructors call `Directory.CreateDirectory` - verify the paths in `MauiProgram.cs` use `FileSystem.AppDataDirectory`. |
| LiteDB `IOException` (file is in use) | A second copy of the app (or an inspection tool) has the file open. Close other instances; never open the on-device file directly - export a copy instead. |
| Typing a bad quantity/price does nothing | `Entry` → `int`/`decimal` bindings keep the old value when parsing fails. That is expected; the validation messages fire on Save. |

### DevFlow troubleshooting

Follow this order - most connection failures are a sequencing problem:

1. **Start the broker first:** `maui devflow broker start`. If it reports the port is in use, a broker is already running - that is fine.
2. **Start the MCP server with the matching platform:** `maui devflow mcp --platform android` (or `windows`, `ios`, `maccatalyst`). A mismatch between this flag and the target you launch means the agent never registers.
3. **Launch the app in DEBUG:** the agent code is wrapped in `#if DEBUG` (step 17), so a Release build will never connect. Use `-c DEBUG`.
4. **Wait for the agent:** `maui devflow agent wait` should print the agent port (for example 9223). If it hangs forever:
   - Confirm steps 1–3 happened **before** the app launched; if not, kill the app and relaunch it.
   - Confirm `builder.AddMauiDevFlowAgent();` really is in `MauiProgram.cs` before `builder.Build()` and the package restored.
   - On Windows, allow the firewall prompt the first time the broker starts.
   - On an Android emulator, remember the emulator reaches your host through its own network bridge - restarting the broker and then the app usually re-establishes the connection.
   - On Mac Catalyst, the app needs the network-server entitlement or the agent cannot open its port.
5. **Inspector does not load:** use the exact port printed by `agent wait` - the URL pattern is `http://localhost:19223/inspector`. Try a different browser profile if a cached page misbehaves.
6. **`storage files list` returns nothing or errors:** the agent connection was lost (re-run `maui devflow agent wait`), the app went to the background (bring it to the foreground), or the root is wrong - use `--root appData`.
7. **`storage files download` fails:** quote the file name exactly as shown in the listing and give an `--output` path you can write to. Re-run the list command first to confirm the agent is still attached.
8. **Exported `sales-aggregates.db` looks stale or empty:** you exported mid-write or before the first tick. Wait for the next `Updated aggregate SQLite database ...` log line, then re-download. Always inspect the downloaded copy, never the live device file.

---

## Student deliverables

Submit the following zip file and other files named as prescribed below. **Do not zip the task files.**

- named full_name.zip - the complete `MauiBackgroundSalesDemo` solution (by way of zipped .git - make sure to use .gitignore and bin/obj folders are not contained in your .git) 
- named task1 - a screenshot of the Dashboard tab showing the seeded demo data
- named task2 - a screenshot of the Sales tab immediately after saving a sale, with the status message visible
- named task3 - a screenshot of the Dashboard tab reflecting that new sale
- named task4 - a screenshot of the Dashboard after a full stop and relaunch, proving the data persisted
- named task5 - the exported `Data/sales-transactions.db` and `Data/sales-aggregates.db` files
- named taskA - the DevFlow `storage files list` JSON output (Task A)
- named taskB - the LiteDB inspection evidence (Task B)
- named taskC - the five SQLite query results (Task C)
- named taskD - the worked aggregation cross-check (Task D)
- named taskE - the before/after background-refresh evidence (Task E)
- named task6 - answers to the twelve reflection questions

Do not submit generated `bin` or `obj` folders.

## Acceptance checklist

- [ ] The project uses .NET 11 and a compatible MAUI 11 workload.
- [ ] `LiteDB` 5.0.21, `Microsoft.Data.Sqlite` 10.0.14, and `CommunityToolkit.Mvvm` 8.4.2 are installed.
- [ ] `ProductSaleTransaction` exposes `Id`, `Timestamp`, `Product`, `Quantity`, `Price`, and a derived `Total`.
- [ ] The three summary model classes exist in `SalesSummaryModels.cs`.
- [ ] `LiteDbSalesStore` stores documents in a typed `sales` collection and ensures the `Timestamp` and `Product` indexes.
- [ ] Both database paths are built from `FileSystem.AppDataDirectory` and use `sales-transactions.db` / `sales-aggregates.db`.
- [ ] Synchronous LiteDB work runs through awaited `Task.Run` calls.
- [ ] Seed data is inserted only when the LiteDB collection is empty.
- [ ] `SqliteSalesAggregateStore` creates `DailySalesSummary`, `ProductSalesSummary`, and `CategorySalesSummary` with `CREATE TABLE IF NOT EXISTS`.
- [ ] All SQLite inserts use parameterized commands, never string-built SQL.
- [ ] `SalesAggregationBackgroundService` extends `BackgroundService` and uses a 15-second `PeriodicTimer`.
- [ ] The stopping token is passed through to every store call and `OperationCanceledException` is handled.
- [ ] `MauiProgram.cs` registers both stores, the background service, both view models, and both pages.
- [ ] The background service is explicitly started with `StartAsync` after `builder.Build()`.
- [ ] `AppShell.xaml` uses a `TabBar` with Dashboard first (default) and Sales second.
- [ ] `DashboardViewModel` reads only from the SQLite aggregate store.
- [ ] `SalesViewModel` validates product, quantity, and price before inserting.
- [ ] Saving a sale inserts into LiteDB and immediately refreshes the SQLite aggregates.
- [ ] `DashboardPage` refreshes its view model in `OnAppearing`.
- [ ] Database work stays in the services - none in page code-behind.
- [ ] Pages receive their view models through constructor injection.
- [ ] The DevFlow agent package and `AddMauiDevFlowAgent()` call are conditional on `DEBUG`.
- [ ] `maui devflow storage files list` shows both database files under the `appData` root.
- [ ] Both database files were exported with the DevFlow storage CLI into `Data/`.
- [ ] Inspection Tasks A–E are complete with the required evidence.
- [ ] The Task D hand calculation matches the `ProductSalesSummary` row exactly.
- [ ] Dashboard data survives a full stop and relaunch on the same target.
- [ ] All twelve reflection questions are answered in your own words.

## Grading rubric (100 points)

| Category | Points |
|---|---|
| Models and LiteDB transactional store (indexes, seeding, threading) | 15 |
| SQLite aggregate store (schema, parameterized SQL, read methods) | 15 |
| Background service (timer, cancellation, DI registration, manual start) | 20 |
| MVVM pages, Shell tabs, validation, and dashboard binding | 15 |
| DevFlow export and database inspection evidence (Tasks A–E) | 15 |
| Reflection answers and overall code quality | 20 |
| **Total** | **100** |
