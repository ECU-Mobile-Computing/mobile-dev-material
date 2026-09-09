# DevFlow Extensions Cheatsheet

This is a quick-reference guide for creating and using DevFlow extensions in MAUI apps. It is intentionally practical: the goal is to help you inspect runtime state, debug app issues, and expose focused diagnostics without turning your app into a giant debugging surface.

## What a DevFlow extension is

A DevFlow extension is code that plugs into a running MAUI app so you can:

- inspect the live UI tree
- read the current page or ViewModel state
- query local app data or embedded databases
- expose custom actions for debugging or automation
- surface structured diagnostics to the DevFlow broker, CLI, or AI tooling

In practice, an extension is usually a small, purpose-built helper that exposes exactly the state you need when debugging a problem.

## When to use an extension

Use an extension when you need to answer questions like:

- What is the current ViewModel for this page?
- What values are currently bound to the UI?
- Is the embedded SQLite database storing the right rows?
- What state does my app service have right now?
- Can I trigger a known debug action without manually clicking through the UI?

If the question is about runtime state, an extension is often the fastest and cleanest answer.

## DevFlow architecture at a glance

The pieces usually involved are:

- MAUI app: the running app that hosts the DevFlow agent
- DevFlow agent: the runtime hooks that expose the app to inspection and automation
- DevFlow broker: local coordinator that routes requests between the app and tooling
- CLI/browser tooling: `maui devflow` commands, browser inspector, and diagnostics UI
- extension surface: your custom code that exposes filtered app state or actions

The key idea is simple: your app exposes state, and DevFlow surfaces it to a developer or automation flow.

## Setup pattern

The exact package and registration APIs may vary by version, but the working pattern is always the same:

1. Add the DevFlow agent to the project.
2. Register the agent in `MauiProgram` in debug builds.
3. Expose a small extension/debug surface for the app state you need.
4. Inspect or invoke it from the running app through DevFlow tools.

Example setup pattern:

```bash
dotnet add package Microsoft.Maui.DevFlow.Agent --prerelease
```

```csharp
using Microsoft.Maui.DevFlow.Agent;

public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        builder.UseMauiApp<App>();

#if DEBUG
        builder.AddMauiDevFlowAgent(options =>
        {
            options.Enabled = true;
            options.CaptureILogger = true;
            options.EnableNetworkMonitoring = true;
        });
#endif

        return builder.Build();
    }
}
```

This is the normal development pattern: keep the runtime hooks in debug builds and keep the extension API focused.

## Create a custom extension

The exact API names depend on your DevFlow version, but the design pattern is always the same:

- create a small service or helper dedicated to debugging
- expose read-only diagnostics or custom actions
- keep the data shape structured and easy to inspect
- avoid surfacing internal implementation details unless they are useful for debugging

Illustrative pattern:

```csharp
public sealed class AppDebugExtension
{
    private readonly IServiceProvider _services;

    public AppDebugExtension(IServiceProvider services)
    {
        _services = services;
    }

    public object InspectCurrentPageState()
    {
        var page = Application.Current?.MainPage;
        var vm = page?.BindingContext;

        return new
        {
            PageType = page?.GetType().FullName,
            ViewModelType = vm?.GetType().FullName,
            ViewModelProperties = vm is null
                ? null
                : vm.GetType()
                    .GetProperties()
                    .Where(p => p.CanRead)
                    .Select(p => new
                    {
                        Name = p.Name,
                        Value = p.GetValue(vm)
                    })
                    .ToList()
        };
    }
}
```

This is a good pattern for a debug-only extension because it tells you exactly what the app believes the current page state is.

## Use the extension in a DevFlow workflow

The developer experience usually looks like this:

1. Start the app under DevFlow.
2. Open the DevFlow UI or CLI tooling.
3. Trigger the extension or inspector action.
4. Review the structured JSON/data response.
5. Fix the page, state, or app logic.
6. Re-run the same checks to confirm the behavior.

Relevant tooling from the MAUI workflow includes commands such as:

```bash
maui devflow ui tree
maui devflow ui diagnostics
maui devflow ui screenshot
```

These commands help you work with the running UI, and your extension should complement them by exposing app-specific state that the visual tree alone cannot explain.

## Example 1: Inspecting a ViewModel

This is the most common debugging scenario. You may have a page where the bindings look wrong, the data is stale, or a command is firing with the wrong values.

A useful extension simply reports the current ViewModel and its public properties.

```csharp
public sealed class ViewModelInspector
{
    public object? Inspect(object? viewModel)
    {
        if (viewModel is null)
            return null;

        var type = viewModel.GetType();

        return new
        {
            Type = type.FullName,
            Properties = type
                .GetProperties()
                .Where(p => p.CanRead)
                .Select(p => new
                {
                    Name = p.Name,
                    Value = p.GetValue(viewModel)
                })
                .ToList()
        };
    }
}
```

Usage pattern:

```csharp
var inspector = new ViewModelInspector();
var state = inspector.Inspect(MyPage.BindingContext);

// Return state to DevFlow or log it to the debug output
Console.WriteLine(System.Text.Json.JsonSerializer.Serialize(state));
```

Why this helps:

- confirms whether the page is bound to the expected ViewModel
- exposes stale values immediately
- helps debug missing data, command state, or sorting/filter issues
- gives a clean, structured snapshot that can be compared before and after a user action

Good question to ask while debugging: “What does the page think the state is right now?”

## Example 2: Inspecting an embedded database table

Another great use for DevFlow extensions is checking data in a local database, especially when the issue is not in the UI but in the persisted state.

Suppose your app stores data in SQLite and you want to inspect a table without opening a separate tool or writing ad hoc scripts.

```csharp
using Microsoft.Data.Sqlite;

public sealed class EmbeddedDbInspector
{
    private readonly SqliteConnection _connection;

    public EmbeddedDbInspector(SqliteConnection connection)
    {
        _connection = connection;
    }

    public object InspectTable(string tableName, int rowLimit = 25)
    {
        var schemaCommand = _connection.CreateCommand();
        schemaCommand.CommandText = "SELECT sql FROM sqlite_master WHERE type = 'table' AND name = @name;";
        schemaCommand.Parameters.AddWithValue("@name", tableName);

        var schema = schemaCommand.ExecuteScalar()?.ToString();

        var rowsCommand = _connection.CreateCommand();
        rowsCommand.CommandText = $"SELECT * FROM {tableName} LIMIT {rowLimit};";

        using var reader = rowsCommand.ExecuteReader();
        var rows = new List<Dictionary<string, object?>>();

        while (reader.Read())
        {
            var row = new Dictionary<string, object?>();
            for (var i = 0; i < reader.FieldCount; i++)
            {
                row[reader.GetName(i)] = reader.IsDBNull(i) ? null : reader.GetValue(i);
            }
            rows.Add(row);
        }

        return new
        {
            Table = tableName,
            Schema = schema,
            Rows = rows
        };
    }
}
```

Usage pattern:

```csharp
var dbInspector = new EmbeddedDbInspector(connection);
var snapshot = dbInspector.InspectTable("Orders");

Console.WriteLine(System.Text.Json.JsonSerializer.Serialize(snapshot));
```

Why this helps:

- confirms whether the app wrote the expected data
- exposes missing rows, wrong values, or stale records
- gives you a snapshot of the exact data the app is using
- helps debug issues that only show up after the page reloads or after a save cycle

This pattern is especially useful when the bug lives in persistence or sync logic rather than rendering.

## A simple “debug snapshot” pattern

If you want a clean extension design, prefer a single debug snapshot method instead of exposing a dozen unrelated functions.

```csharp
public sealed class AppDebugSnapshot
{
    private readonly ViewModelInspector _viewModelInspector;
    private readonly EmbeddedDbInspector _dbInspector;

    public AppDebugSnapshot(ViewModelInspector viewModelInspector, EmbeddedDbInspector dbInspector)
    {
        _viewModelInspector = viewModelInspector;
        _dbInspector = dbInspector;
    }

    public object Capture(string tableName)
    {
        var page = Application.Current?.MainPage;
        var vm = page?.BindingContext;

        return new
        {
            CurrentPage = page?.GetType().FullName,
            ViewModel = _viewModelInspector.Inspect(vm),
            Database = _dbInspector.InspectTable(tableName)
        };
    }
}
```

This is a good pattern because it makes the extension easy to reason about:

- one purpose
- one structured result
- one debug snapshot to inspect

## Best practices for extension development

Keep these rules in mind when building DevFlow extensions:

- Keep the extension focused on one problem or workflow.
- Return structured data instead of raw strings when possible.
- Prefer debug-only registration for sensitive or noisy features.
- Check current page and current binding context before exposing state.
- Keep database inspection safe and bounded; use `LIMIT` or row caps when possible.
- Log clear metadata like page name, table name, or action name.
- Avoid leaking secrets, tokens, or user data in debug output.

## Quick troubleshooting checklist

If your extension is not giving useful output:

- confirm the app is actually running under the DevFlow agent
- ensure the extension is registered in debug mode
- verify the page or ViewModel is not null at the time of inspection
- confirm the database connection is open and the table really exists
- inspect the current route or page before reading state
- compare the state before and after the user action that triggers the bug

## Short reference

Use this as a quick cheat sheet:

- Want to inspect current page state? Expose the `BindingContext` and relevant properties.
- Want to inspect UI state? Use DevFlow tree and diagnostics tooling alongside your extension.
- Want to inspect the DB? Query the schema and a bounded sample of rows.
- Want a reusable debug surface? Build a single snapshot helper, not a dozen unrelated actions.
- Want to keep things maintainable? Keep the extension debug-only and structured.

## Summary

The best DevFlow extension is usually a tiny, targeted, debug-focused helper. It should provide the answers that are hard to get from the visual tree alone: current ViewModel state, persisted data, and app-specific runtime conditions.

If you keep the extension narrow and structured, it will become one of the most useful tools in your MAUI debugging workflow.
