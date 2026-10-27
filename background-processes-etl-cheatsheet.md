# .NET 11 MAUI Background Processes and Device ETL Cheat Sheet

This cheatsheet is a practical reference for background processing in .NET 11 MAUI and for creating a small local ETL pipeline on the device. The goal is to keep the UI responsive while moving heavier work into background tasks, then storing a processed summary in SQLite so charts and screens can read fast, simple data.

## 1. The core rule: UI work stays on the UI thread

In MAUI, the app UI should remain responsive. Do not perform database queries, filtering, aggregation, or heavy transforms directly on the page or view model while the user is interacting with controls.

Good pattern:

- UI thread: render labels, buttons, lists, and navigation
- background task: load data, compute totals, aggregate time series, and update local SQLite tables
- UI thread again: bind the processed result back to the page

```csharp
public async Task RefreshSummaryAsync()
{
    var summary = await _etlService.RunDeviceEtlAsync();

    await MainThread.InvokeOnMainThreadAsync(() =>
    {
        Summary = summary;
    });
}
```

Use `MainThread.InvokeOnMainThreadAsync(...)` or `Dispatcher.DispatchAsync(...)` when updating UI-bound properties after background work completes.

## 2. Common background patterns in MAUI

### Async operations for local work

Use `async` and `await` for file I/O, SQLite reads, LiteDB reads, and local aggregation.

```csharp
public async Task<List<ReadingRecord>> LoadLocalDataAsync(CancellationToken cancellationToken)
{
    var records = await _liteDb.GetCollection<ReadingRecord>()
        .FindAll()
        .ToListAsync(cancellationToken);

    return records;
}
```

### Task.Run for CPU-bound work

Use `Task.Run()` only for work that is CPU-heavy and not tied to the UI. Typical examples include grouping, sorting, indexing, or large transforms.

```csharp
public async Task<List<DailySummary>> AggregateReadingsAsync(List<ReadingRecord> records)
{
    return await Task.Run(() =>
    {
        return records
            .GroupBy(r => r.Timestamp.Date)
            .Select(g => new DailySummary
            {
                Date = g.Key,
                Total = g.Sum(x => x.Value),
                Average = g.Average(x => x.Value)
            })
            .OrderBy(x => x.Date)
            .ToList();
    });
}
```

### Cancellation tokens

A proper background task should accept a `CancellationToken` so the work can stop cleanly when the app is backgrounded or the user navigates away.

```csharp
public async Task RunAsync(CancellationToken cancellationToken)
{
    var rawData = await _liteDbService.LoadAsync(cancellationToken);
    var aggregated = await AggregateReadingsAsync(rawData, cancellationToken);
    await _sqliteService.WriteSummaryAsync(aggregated, cancellationToken);
}
```

### App lifecycle awareness

If the app is being paused or stopped, do not assume the work will continue indefinitely. Background jobs should check cancellation and avoid writing UI state to stale pages or old view models.

Typical lifecycle points:

- `OnAppearing()` for page-level refresh
- `OnDisappearing()` for cleanup
- `App.OnSleep()` and `App.OnResume()` for app-level state tracking
- `BackgroundService` or platform-specific background work only when the app truly needs long-running background execution

## 3. Choosing the right approach

Use this quick decision guide:

- Simple local database read or write: `async`/`await`
- Data transformation or aggregation: `Task.Run()` if CPU-heavy
- Background sync or long-running device work: platform background job or service, not direct UI thread work
- UI binding updates: switch back to the main thread before updating controls

## 4. Device ETL pattern: LiteDB to SQLite for visual data

A common MAUI pattern is to keep raw operational data in LiteDB and then create a simplified SQLite summary database for charts, cards, and dashboard widgets.

This pattern keeps the source data complete and durable while making the visual layer fast and easy to query.

### Flow

```text
LiteDB (raw operational data)
        |
        v
Read, clean, and convert data
        |
        v
Aggregate by day, hour, category, or tag
        |
        v
Write optimized SQLite snapshot tables
        |
        v
App pages bind to SQLite summary tables
```

### Why this is useful

- LiteDB is a good place for structured raw records on device
- SQLite is often easier and faster for chart queries and dashboard reads
- the app can render visuals without scanning large raw datasets every time
- the transformed tables can be refreshed on a schedule or after data changes

## 5. Example: ETL service for a device dashboard

Below is a practical pattern for reading LiteDB records, aggregating them, and writing the result to SQLite for UI consumption.

```csharp
public sealed class DeviceEtlService
{
    private readonly LiteDatabase _liteDb;
    private readonly SqliteConnection _sqlite;

    public DeviceEtlService(LiteDatabase liteDb, SqliteConnection sqlite)
    {
        _liteDb = liteDb;
        _sqlite = sqlite;
    }

    public async Task RunAsync(CancellationToken cancellationToken)
    {
        var raw = await LoadRawDataAsync(cancellationToken);
        var summary = AggregateByDay(raw);
        await WriteSummaryAsync(summary, cancellationToken);
    }

    private Task<List<ReadingRecord>> LoadRawDataAsync(CancellationToken cancellationToken)
    {
        return Task.Run(() =>
        {
            var collection = _liteDb.GetCollection<ReadingRecord>("readings");
            return collection.FindAll().ToList();
        }, cancellationToken);
    }

    private static List<DailySummary> AggregateByDay(List<ReadingRecord> records)
    {
        return records
            .GroupBy(r => r.Timestamp.Date)
            .Select(g => new DailySummary
            {
                Date = g.Key,
                Total = g.Sum(x => x.Value),
                Average = g.Average(x => x.Value),
                Count = g.Count()
            })
            .OrderBy(x => x.Date)
            .ToList();
    }

    private async Task WriteSummaryAsync(List<DailySummary> rows, CancellationToken cancellationToken)
    {
        await using var command = _sqlite.CreateCommand();
        command.CommandText = "DELETE FROM DailySummary";
        await command.ExecuteNonQueryAsync(cancellationToken);

        foreach (var row in rows)
        {
            await using var insert = _sqlite.CreateCommand();
            insert.CommandText = @"
                INSERT INTO DailySummary (DateKey, Total, Average, Count)
                VALUES (@DateKey, @Total, @Average, @Count)";

            insert.Parameters.AddWithValue("@DateKey", row.Date.ToString("yyyy-MM-dd"));
            insert.Parameters.AddWithValue("@Total", row.Total);
            insert.Parameters.AddWithValue("@Average", row.Average);
            insert.Parameters.AddWithValue("@Count", row.Count);

            await insert.ExecuteNonQueryAsync(cancellationToken);
        }
    }
}
```

This is the classic pattern:

1. read raw records from LiteDB
2. aggregate them into a dashboard-friendly shape
3. write the aggregate table into SQLite
4. let the page bind to the SQLite result set

## 6. SQLite table for visuals

Your SQLite database should be optimized for the screen or chart you are showing, not a raw copy of the LiteDB table.

```sql
CREATE TABLE DailySummary (
    Id INTEGER PRIMARY KEY AUTOINCREMENT,
    DateKey TEXT NOT NULL,
    Total REAL NOT NULL,
    Average REAL NOT NULL,
    Count INTEGER NOT NULL
);
```

This table is much easier for the app to query than a large raw table with detailed sensor or transaction rows.

Example of a query used by a chart or dashboard view model:

```csharp
var rows = await _sqliteConnection.QueryAsync<DailySummary>(
    "SELECT DateKey, Total, Average, Count FROM DailySummary ORDER BY DateKey DESC");
```

Then the UI can bind directly to the results.

## 7. View Model pattern for a chart-driven app

The page should not do ETL work. It should request data and then bind the result to a visual collection.

```csharp
public sealed class DashboardViewModel : ObservableObject
{
    private readonly DeviceEtlService _etlService;

    public ObservableCollection<DailySummary> DailySummaries { get; } = new();

    public DashboardViewModel(DeviceEtlService etlService)
    {
        _etlService = etlService;
    }

    public async Task RefreshAsync()
    {
        await _etlService.RunAsync(CancellationToken.None);

        var rows = await LoadSummaryRowsFromSqliteAsync();

        DailySummaries.Clear();
        foreach (var row in rows)
            DailySummaries.Add(row);
    }
}
```

This separation is important:

- `DeviceEtlService`: reads -> transforms -> writes
- `DashboardViewModel`: loads chart-ready rows and exposes them to the page
- `Page`: binds the data and displays it in a `CollectionView`, `Chart`, or card layout

## 8. Practical rules for device ETL

- Keep raw source data in LiteDB or another local store
- Write only summarized data into SQLite when it is needed for screen performance
- Refresh only when needed, not on every keystroke
- Avoid reading large raw data sets on the UI thread
- Use a timestamped, idempotent update strategy so the summary table can be refreshed safely
- Use a consistent schema for the visual database
- Prefer database queries that return small result sets for the dashboard

## 9. Example ETL update strategy

A simple and reliable strategy is to replace the summary table each time the ETL runs.

```csharp
await using var cmd = _sqlite.CreateCommand();
cmd.CommandText = "DELETE FROM DailySummary";
await cmd.ExecuteNonQueryAsync();
```

This is easy to reason about and works well when the dataset is small to medium sized and the summary is rebuilt from local source data.

For larger workloads, consider:

- upsert by date key
- incremental updates only for changed records
- a watermark timestamp from the last successful ETL
- a queue or background worker triggered after data changes

## 10. When to use a background worker

If your app performs ETL on a schedule or when new data arrives, a background worker can help:

- app opens and new data appears
- local JSON or LiteDB records change in the background
- dashboard summary must refresh without blocking user actions
- periodic aggregation should happen even when the user is not in a specific page

In MAUI, keep this logic moderate and device-aware. Do not rely on long-running background tasks for work that can be triggered on demand. For true long-running background execution, use platform-appropriate APIs such as Android background work or OS-level job scheduling.

## 11. Quick reference

### UI vs background

| Task | Where it belongs |
|---|---|
| Button clicks and page updates | UI thread |
| Loading raw records from LiteDB | background task |
| Aggregating by day/hour/category | background task |
| Writing SQLite summary tables | background task |
| Updating `ObservableCollection` or a chart | UI thread after background work |

### Typical commands and APIs

```csharp
await Task.Run(() => { /* CPU-bound aggregation */ });
await MainThread.InvokeOnMainThreadAsync(() => { /* update UI */ });
using CancellationTokenSource cts = new();
```

### Typical architecture

```text
Page -> ViewModel -> ETL service -> LiteDB -> aggregate -> SQLite -> visual chart
```

## 12. Takeaway

The main idea is simple:

- keep the page and user interface lightweight
- process data in the background
- store a summarized, dashboard-friendly result in SQLite
- let the UI bind to those small tables quickly and reliably

That creates a MAUI app that stays responsive while still giving the user rich, up-to-date charts and summaries based on local device data.
