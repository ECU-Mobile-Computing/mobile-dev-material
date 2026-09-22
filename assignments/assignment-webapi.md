# Assignment: Course Assignment Tracker with .NET MAUI and ASP.NET Core

## Overview

Build a .NET 11 Android application that stores course assignments in a remote SQLite database. The app opens on a list and lets a user retrieve, add, update, and delete assignments through an ASP.NET Core Web API.

The SQLite file belongs to the API. The MAUI app must never open or modify that file directly.

This guided assignment is for students who are new to C#, mobile programming, HTTP APIs, and .NET MAUI. Complete the sections in order and verify each checkpoint before continuing.

### Learning outcomes

By the end, you should be able to:

- explain how a MAUI app, Web API, and database work together
- implement asynchronous CRUD endpoints with ASP.NET Core controllers
- persist records with Entity Framework Core and SQLite
- send JSON through a typed `HttpClient` service
- use MVVM Toolkit, dependency injection, commands, and compiled bindings
- navigate between list, add, and edit pages with Shell
- handle validation, missing records, timeouts, and network failures
- expose a local API temporarily with Microsoft Dev Tunnels
- verify API behavior manually and inspect a running app with DevFlow

## Architecture

```text
Android MAUI app
	   |
	   | HTTPS + JSON
	   v
Microsoft Dev Tunnel
	   |
	   | forwards to http://localhost:5050 on the computer
	   v
ASP.NET Core Web API
	   |
	   | Entity Framework Core
	   v
courseassignments.db (SQLite)
```

The solution has three projects:

| Project | Responsibility |
| --- | --- |
| `CourseAssignments.Contracts` | Request and response models shared by client and server |
| `CourseAssignments.Api` | HTTP endpoints, validation, EF Core, and SQLite |
| `CourseAssignments.App` | Android UI, navigation, view models, and HTTP client |

Every assignment has four fields:

| Field | Type | Purpose |
| --- | --- | --- |
| `AssignmentId` | `int` | Database-generated identifier |
| `Title` | `string` | Assignment name, 2-100 characters |
| `DueDate` | `DateTime` | Date the work is due |
| `IsCompleted` | `bool` | Whether the work is complete |

## Android localhost and networking

Networking is one of the most common sources of confusion in mobile development. The app and API are separate processes, often running on separate operating systems.

### Why `localhost` fails from Android

`localhost` and `127.0.0.1` always mean **the machine making the request**:

- In a desktop browser, `http://localhost:5050` means port 5050 on the development computer.
- In an Android emulator, `http://localhost:5050` means port 5050 inside the emulator.
- On a physical Android phone, it means port 5050 on the phone.

The API is not running inside Android, so an Android request to `localhost:5050` normally fails with connection refused.

### Other development connection choices

The standard Android Emulator commonly maps `10.0.2.2` to the host computer's loopback adapter. Other emulators can use a different special address. A physical phone cannot use `10.0.2.2`; it usually needs the computer's LAN address, matching Wi-Fi, firewall permission, and an API listening on a non-loopback interface.

Local HTTP and HTTPS introduce more challenges:

- Android can block cleartext HTTP unless the app explicitly allows it.
- A development HTTPS certificate trusted by the computer may not be trusted by Android.
- Binding the API only to `localhost` prevents other LAN devices from reaching it.
- Firewalls, VPNs, guest Wi-Fi isolation, and changing IP addresses can interrupt LAN access.
- Android manifest internet permission is required.
- A URL that works in a desktop browser does not prove it works from the emulator.

### Why this assignment uses a Dev Tunnel

Microsoft Dev Tunnels gives Android a public HTTPS URL and forwards requests to the API's local HTTP port. This avoids emulator-specific host aliases, LAN addresses, cleartext configuration, and direct trust of the ASP.NET development certificate.

The tunnel does not remove every failure point. The API and tunnel processes must both remain running, the temporary URL can change, internet access is required, and anonymous access must be treated carefully. You will test each connection separately:

1. Desktop to local API: `http://localhost:5050`
2. Desktop to public tunnel: `https://...devtunnels.ms`
3. Android app to public tunnel

This three-step check identifies which networking layer is failing.

## Prerequisites

You need:

- an installed .NET 11 prerelease SDK and compatible MAUI 11 workload
- Microsoft MAUI CLI and the VS Code .NET MAUI extension
- an Android emulator already created with the MAUI CLI
- a Microsoft or GitHub account for Dev Tunnels

The emulator is already configured for this course. Do not reinstall Android tooling or create another emulator. When needed, start the existing emulator:

```console
maui android emulator list
maui android emulator start
```

> .NET 11 is prerelease software for this assignment. Use an SDK version actually installed on your computer.

## 1. Create the solution

```console
mkdir CourseAssignments
cd CourseAssignments
dotnet new globaljson --sdk-version YOUR_INSTALLED_11_SDK_VERSION --roll-forward latestPatch
```

Replace the placeholder with the complete version from `dotnet --list-sdks`. Add `"allowPrerelease": true` inside the generated `sdk` object.

```console
dotnet new sln --name CourseAssignments --format slnx
dotnet new classlib --name CourseAssignments.Contracts --output src/CourseAssignments.Contracts --framework net11.0
dotnet new webapi --name CourseAssignments.Api --output src/CourseAssignments.Api --framework net11.0 --use-controllers
dotnet new maui --name CourseAssignments.App --output src/CourseAssignments.App --framework net11.0

dotnet sln CourseAssignments.slnx add src/CourseAssignments.Contracts/CourseAssignments.Contracts.csproj
dotnet sln CourseAssignments.slnx add src/CourseAssignments.Api/CourseAssignments.Api.csproj
dotnet sln CourseAssignments.slnx add src/CourseAssignments.App/CourseAssignments.App.csproj
dotnet add src/CourseAssignments.Api/CourseAssignments.Api.csproj reference src/CourseAssignments.Contracts/CourseAssignments.Contracts.csproj
dotnet add src/CourseAssignments.App/CourseAssignments.App.csproj reference src/CourseAssignments.Contracts/CourseAssignments.Contracts.csproj
```
Add Packages

```console
dotnet add src/CourseAssignments.Api/CourseAssignments.Api.csproj package Microsoft.EntityFrameworkCore.Sqlite --prerelease
dotnet add src/CourseAssignments.Api/CourseAssignments.Api.csproj package Microsoft.EntityFrameworkCore.Design --prerelease
dotnet add src/CourseAssignments.App/CourseAssignments.App.csproj package CommunityToolkit.Mvvm
dotnet add src/CourseAssignments.App/CourseAssignments.App.csproj package Microsoft.Extensions.Http

```

Install the Entity Framework tool
```console
dotnet tool install --global dotnet-ef --prerelease 
```
> Note: If you receive a message stating that the tool is already installed then make sure it is updated using the following command <br/>`dotnet tool update --global dotnet-ef --prerelease`


All EF Core packages and `dotnet-ef` must use major version 11. Verify with `dotnet ef --version` and `dotnet list src/CourseAssignments.Api/CourseAssignments.Api.csproj package`.

## 2. Create the shared contracts

Create `src/CourseAssignments.Contracts/Models/AssignmentDto.cs`:

```csharp
namespace CourseAssignments.Contracts.Models;

public sealed record AssignmentDto(
	int AssignmentId,
	string Title,
	DateTime DueDate,
	bool IsCompleted);
```

Create `src/CourseAssignments.Contracts/Models/UpsertAssignmentRequest.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace CourseAssignments.Contracts.Models;

public sealed class UpsertAssignmentRequest
{
	[Required]
	[StringLength(100, MinimumLength = 2)]
	public string Title { get; init; } = string.Empty;

	public DateTime DueDate { get; init; }

	public bool IsCompleted { get; init; }
}
```

The response includes the database ID. The create/update request does not let a client choose that ID.

## 3. Add SQLite and Entity Framework Core

Create `src/CourseAssignments.Api/Entities/CourseAssignment.cs`:

```csharp
namespace CourseAssignments.Api.Entities;

public sealed class CourseAssignment
{
	public int AssignmentId { get; set; }
	public required string Title { get; set; }
	public DateTime DueDate { get; set; }
	public bool IsCompleted { get; set; }
}
```

Create `src/CourseAssignments.Api/Data/AssignmentsDbContext.cs`:

```csharp
using CourseAssignments.Api.Entities;
using Microsoft.EntityFrameworkCore;

namespace CourseAssignments.Api.Data;

public sealed class AssignmentsDbContext(DbContextOptions<AssignmentsDbContext> options)
	: DbContext(options)
{
	public DbSet<CourseAssignment> Assignments => Set<CourseAssignment>();

	protected override void OnModelCreating(ModelBuilder modelBuilder)
	{
		modelBuilder.Entity<CourseAssignment>(entity =>
		{
			entity.HasKey(item => item.AssignmentId);
			entity.Property(item => item.Title).HasMaxLength(100).IsRequired();
		});
	}
}
```

Replace `src/CourseAssignments.Api/appsettings.json`:

```json
{
  "ConnectionStrings": {
	"AssignmentsDatabase": "Data Source=courseassignments.db"
  },
  "Logging": {
	"LogLevel": {
	  "Default": "Information",
	  "Microsoft.AspNetCore": "Warning"
	}
  },
  "AllowedHosts": "*"
}
```

Replace `src/CourseAssignments.Api/Program.cs`:

```csharp
using CourseAssignments.Api.Data;
using CourseAssignments.Api.Endpoints;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);
var connectionString = builder.Configuration.GetConnectionString("AssignmentsDatabase")
	?? throw new InvalidOperationException("AssignmentsDatabase is not configured.");

builder.Services.AddValidation();
builder.Services.AddProblemDetails();
builder.Services.AddDbContext<AssignmentsDbContext>(options =>
	options.UseSqlite(connectionString));

builder.Services.AddCors();

var app = builder.Build();
app.UseExceptionHandler();

app.UseCors(builder => builder
	.AllowAnyOrigin()
	.AllowAnyMethod()
	.AllowAnyHeader());

app.MapAssignmentsEndpoints();

await using (var scope = app.Services.CreateAsyncScope())
{
	var database = scope.ServiceProvider.GetRequiredService<AssignmentsDbContext>();
	await database.Database.MigrateAsync();
}

await app.RunAsync();
```

The API uses local HTTP for this lab. The Dev Tunnel supplies the external HTTPS endpoint. This is a development arrangement, not a production deployment.

## 4. Implement API CRUD operations

Create `src/CourseAssignments.Api/Endpoints/AssignmentEndpoints.cs`:

```csharp
using CourseAssignments.Api.Data;
using CourseAssignments.Api.Entities;
using CourseAssignments.Contracts.Models;
using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.EntityFrameworkCore;

namespace CourseAssignments.Api.Endpoints;

public static class AssignmentsEndpoints
{
	public static WebApplication MapAssignmentsEndpoints(this WebApplication app)
	{
		var group = app.MapGroup("/api/assignments");
		group.MapGet("", GetAllAsync);
		group.MapGet("/{id:int}", GetByIdAsync).WithName(nameof(GetByIdAsync));
		group.MapPost("", CreateAsync);
		group.MapPut("/{id:int}", UpdateAsync);
		group.MapDelete("/{id:int}", DeleteAsync);
		return app;
	}

	private static async Task<Ok<List<AssignmentDto>>> GetAllAsync(
		AssignmentsDbContext database,
		CancellationToken cancellationToken)
	{
		var items = await database.Assignments.AsNoTracking()
			.OrderBy(item => item.DueDate)
			.Select(item => new AssignmentDto(
				item.AssignmentId, item.Title, item.DueDate, item.IsCompleted))
			.ToListAsync(cancellationToken);
		return TypedResults.Ok(items);
	}

	private static async Task<Results<Ok<AssignmentDto>, NotFound>> GetByIdAsync(
		int id,
		AssignmentsDbContext database,
		CancellationToken cancellationToken)
	{
		var item = await database.Assignments.AsNoTracking()
			.Where(item => item.AssignmentId == id)
			.Select(item => new AssignmentDto(
				item.AssignmentId, item.Title, item.DueDate, item.IsCompleted))
			.SingleOrDefaultAsync(cancellationToken);
		return item is null ? TypedResults.NotFound() : TypedResults.Ok(item);
	}

	private static async Task<Results<Created<AssignmentDto>, ValidationProblem>> CreateAsync(
		UpsertAssignmentRequest request,
		AssignmentsDbContext database,
		CancellationToken cancellationToken)
	{
		var validationProblem = ValidateDueDate(request);
		if (validationProblem is not null)
			return validationProblem;

		var entity = new CourseAssignment
		{
			Title = request.Title.Trim(),
			DueDate = request.DueDate.Date,
			IsCompleted = request.IsCompleted
		};
		database.Assignments.Add(entity);
		await database.SaveChangesAsync(cancellationToken);

		var result = ToDto(entity);
		return TypedResults.Created($"/api/assignments/{entity.AssignmentId}", result);
	}

	private static async Task<Results<NoContent, NotFound, ValidationProblem>> UpdateAsync(
		int id,
		UpsertAssignmentRequest request,
		AssignmentsDbContext database,
		CancellationToken cancellationToken)
	{
		var validationProblem = ValidateDueDate(request);
		if (validationProblem is not null)
			return validationProblem;

		var entity = await database.Assignments.FindAsync([id], cancellationToken);
		if (entity is null)
			return TypedResults.NotFound();

		entity.Title = request.Title.Trim();
		entity.DueDate = request.DueDate.Date;
		entity.IsCompleted = request.IsCompleted;
		await database.SaveChangesAsync(cancellationToken);
		return TypedResults.NoContent();
	}

	private static async Task<Results<NoContent, NotFound>> DeleteAsync(
		int id,
		AssignmentsDbContext database,
		CancellationToken cancellationToken)
	{
		var entity = await database.Assignments.FindAsync([id], cancellationToken);
		if (entity is null)
			return TypedResults.NotFound();

		database.Assignments.Remove(entity);
		await database.SaveChangesAsync(cancellationToken);
		return TypedResults.NoContent();
	}

	private static ValidationProblem? ValidateDueDate(UpsertAssignmentRequest request) =>
		request.DueDate == default
			? TypedResults.ValidationProblem(new Dictionary<string, string[]>
			{
				[nameof(request.DueDate)] = ["A due date is required."]
			})
			: null;

	private static AssignmentDto ToDto(CourseAssignment item) =>
		new(item.AssignmentId, item.Title, item.DueDate, item.IsCompleted);
}
```


## 5. Create and test the database

From the solution directory, create the migration:

```console
dotnet ef migrations add InitialCreate --project src/CourseAssignments.Api/CourseAssignments.Api.csproj --startup-project src/CourseAssignments.Api/CourseAssignments.Api.csproj --output-dir Data/Migrations
```

Start the API on a predictable local port:

```console
dotnet run --project src/CourseAssignments.Api/CourseAssignments.Api.csproj --urls http://localhost:5050
```

The API applies pending migrations at startup. Confirm that `courseassignments.db` appears in the API project directory. Keep this terminal running.

### Checkpoint A: local API

Open <http://localhost:5050/api/assignments>. A new database should return:

```json
[]
```

Create `requests.http` at the solution root:

```http
@baseUrl = http://localhost:5050
@assignmentId = 1

### Retrieve all
GET {{baseUrl}}/api/assignments
Accept: application/json

### Create
POST {{baseUrl}}/api/assignments
Content-Type: application/json

{
  "title": "Build the MAUI list page",
  "dueDate": "2026-10-15T00:00:00",
  "isCompleted": false
}

### Retrieve one
GET {{baseUrl}}/api/assignments/{{assignmentId}}
Accept: application/json

### Update
PUT {{baseUrl}}/api/assignments/{{assignmentId}}
Content-Type: application/json

{
  "title": "Build and verify the MAUI list page",
  "dueDate": "2026-10-16T00:00:00",
  "isCompleted": true
}

### Delete
DELETE {{baseUrl}}/api/assignments/{{assignmentId}}

### Invalid title: expect 400
POST {{baseUrl}}/api/assignments
Content-Type: application/json

{
  "title": "",
  "dueDate": "2026-10-15T00:00:00",
  "isCompleted": false
}

### Missing ID: expect 404
GET {{baseUrl}}/api/assignments/999999
Accept: application/json
```

Use the VS Code REST Client extension or another HTTP client. Send requests one at a time. Replace `@assignmentId` if the generated ID is not 1.

| Operation | Expected response |
| --- | ---: |
| Retrieve all/one | `200 OK` |
| Create | `201 Created` |
| Update/delete | `204 No Content` |
| Invalid title | `400 Bad Request` |
| Missing ID | `404 Not Found` |

## 6. Install and run Microsoft Dev Tunnels

Install only the Dev Tunnels CLI for your host operating system. These are not Android setup steps.

### Windows

```powershell
winget install Microsoft.devtunnel
```

Use `winget upgrade Microsoft.devtunnel` when it is already installed.

### macOS

```bash
brew install --cask devtunnel
```

Without Homebrew:

```bash
curl -sL https://aka.ms/DevTunnelCliInstall | bash
```

### Linux

```bash
curl -sL https://aka.ms/DevTunnelCliInstall | bash
```

### Sign in and host the API

These steps are the same on all three host systems:

```console
devtunnel --version
devtunnel user login
```

Use `devtunnel user login -g` for GitHub or add `-d` for device-code login. Keep the API running, open another terminal, and run:

```console
devtunnel host -p 5050 --protocol http --allow-anonymous
```

Copy the printed HTTPS URL, which resembles:

```text
https://example-5050.region.devtunnels.ms
```

Open `https://YOUR-TUNNEL-URL/api/assignments` in a desktop browser. Then change `@baseUrl` in `requests.http` to that URL and repeat the GET request.

### Checkpoint B: tunnel

Do not continue until:

- the local URL returns `200 OK`
- the tunnel URL returns `200 OK`
- the API and tunnel terminals remain running

Press `Ctrl+C` to stop the tunnel after the lab. A temporary tunnel is deleted when its host process exits, so its URL can change next time.

> `--allow-anonymous` means anyone who discovers the URL can call this API. Use fake classroom data, never expose secrets, and stop the tunnel after testing. Dev Tunnels is a preview development service, not production hosting.

## 7. Configure Android and DevFlow

Create `src/CourseAssignments.App/Models/ApiSettings.cs`:

```csharp
namespace CourseAssignments.App.Models;

public static class ApiSettings
{
	public const string BaseUrl = "https://YOUR-TUNNEL-URL";
}
```

Replace the placeholder and keep the trailing slash. If the temporary tunnel restarts with a different URL, update this value and redeploy the app.

Ensure `src/CourseAssignments.App/Platforms/Android/AndroidManifest.xml` contains this child of `<manifest>`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

Do not enable cleartext traffic. The Android app uses the tunnel's HTTPS endpoint.

Add these properties to the MAUI project's main `PropertyGroup` if missing:

```xml
<EnablePreviewFeatures>true</EnablePreviewFeatures>
<MauiXamlInflator>SourceGen</MauiXamlInflator>
```

## 8. Create the API service

Create `src/CourseAssignments.App/Services/AssignmentApiService.cs`:

```csharp
using System.Net;
using System.Net.Http.Json;
using CourseAssignments.Contracts.Models;

namespace CourseAssignments.App.Services;

public sealed class AssignmentApiService(HttpClient httpClient)
{
	public async Task<IReadOnlyList<AssignmentDto>> GetAllAsync(
		CancellationToken cancellationToken = default) =>
		await httpClient.GetFromJsonAsync<List<AssignmentDto>>(
			"api/assignments", cancellationToken) ?? [];

	public async Task<AssignmentDto?> GetByIdAsync(
		int id, CancellationToken cancellationToken = default)
	{
		using var response = await httpClient.GetAsync(
			$"api/assignments/{id}", cancellationToken);
		if (response.StatusCode == HttpStatusCode.NotFound)
			return null;

		await EnsureSuccessAsync(response, cancellationToken);
		return await response.Content.ReadFromJsonAsync<AssignmentDto>(cancellationToken);
	}

	public async Task<AssignmentDto> CreateAsync(
		UpsertAssignmentRequest request, CancellationToken cancellationToken = default)
	{
		using var response = await httpClient.PostAsJsonAsync(
			"api/assignments", request, cancellationToken);
		await EnsureSuccessAsync(response, cancellationToken);
		return await response.Content.ReadFromJsonAsync<AssignmentDto>(cancellationToken)
			?? throw new HttpRequestException("The API returned an empty create response.");
	}

	public async Task UpdateAsync(
		int id, UpsertAssignmentRequest request, CancellationToken cancellationToken = default)
	{
		using var response = await httpClient.PutAsJsonAsync(
			$"api/assignments/{id}", request, cancellationToken);
		await EnsureSuccessAsync(response, cancellationToken);
	}

	public async Task DeleteAsync(int id, CancellationToken cancellationToken = default)
	{
		using var response = await httpClient.DeleteAsync(
			$"api/assignments/{id}", cancellationToken);
		await EnsureSuccessAsync(response, cancellationToken);
	}

	private static async Task EnsureSuccessAsync(
		HttpResponseMessage response, CancellationToken cancellationToken)
	{
		if (response.IsSuccessStatusCode)
			return;

		var body = await response.Content.ReadAsStringAsync(cancellationToken);
		throw new HttpRequestException(
			$"API request failed with {(int)response.StatusCode} {response.ReasonPhrase}. {body}");
	}
}
```

The service owns URLs, JSON, and HTTP status handling. View models should not repeat those details.

## 9. Create the view models

Create `src/CourseAssignments.App/ViewModels/AssignmentsViewModel.cs`:

```csharp
using System.Collections.ObjectModel;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using CourseAssignments.App.Services;
using CourseAssignments.App.Views;
using CourseAssignments.Contracts.Models;

namespace CourseAssignments.App.ViewModels;

public partial class AssignmentsViewModel(AssignmentApiService api) : ObservableObject
{
	public ObservableCollection<AssignmentDto> Assignments { get; } = [];

	[ObservableProperty]
	private bool isBusy;

	[ObservableProperty]
	private string errorMessage = string.Empty;

	public bool HasError => !string.IsNullOrWhiteSpace(ErrorMessage);

	partial void OnErrorMessageChanged(string value) => OnPropertyChanged(nameof(HasError));

	[RelayCommand]
	public async Task LoadAsync()
	{
		if (IsBusy)
			return;

		try
		{
			IsBusy = true;
			ErrorMessage = string.Empty;
			var items = await api.GetAllAsync();
			Assignments.Clear();
			foreach (var item in items)
				Assignments.Add(item);
		}
		catch (Exception exception) when (exception is HttpRequestException or TaskCanceledException)
		{
			ErrorMessage = "Cannot reach the API. Check the API, tunnel, and app URL.";
		}
		finally
		{
			IsBusy = false;
		}
	}

	[RelayCommand]
	private static Task AddAsync() => Shell.Current.GoToAsync(nameof(AddAssignmentPage));

	[RelayCommand]
	private static Task EditAsync(AssignmentDto? item) =>
		item is null
			? Task.CompletedTask
			: Shell.Current.GoToAsync($"{nameof(EditAssignmentPage)}?id={item.AssignmentId}");

	[RelayCommand]
	private async Task DeleteAsync(AssignmentDto? item)
	{
		if (item is null || IsBusy)
			return;

		try
		{
			IsBusy = true;
			ErrorMessage = string.Empty;
			await api.DeleteAsync(item.AssignmentId);
			Assignments.Remove(item);
		}
		catch (Exception exception) when (exception is HttpRequestException or TaskCanceledException)
		{
			ErrorMessage = "The assignment could not be deleted. Refresh and try again.";
		}
		finally
		{
			IsBusy = false;
		}
	}
}
```

Create `src/CourseAssignments.App/ViewModels/AddAssignmentViewModel.cs`:

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using CourseAssignments.App.Services;
using CourseAssignments.Contracts.Models;

namespace CourseAssignments.App.ViewModels;

public partial class AddAssignmentViewModel(AssignmentApiService api) : ObservableObject
{
	[ObservableProperty] private string title = string.Empty;
	[ObservableProperty] private DateTime dueDate = DateTime.Today.AddDays(7);
	[ObservableProperty] private bool isCompleted;
	[ObservableProperty] private bool isBusy;
	[ObservableProperty] private string errorMessage = string.Empty;

	public bool HasError => !string.IsNullOrWhiteSpace(ErrorMessage);
	partial void OnErrorMessageChanged(string value) => OnPropertyChanged(nameof(HasError));

	[RelayCommand]
	private async Task SaveAsync()
	{
		if (IsBusy)
			return;

		if (string.IsNullOrWhiteSpace(Title) || Title.Trim().Length is < 2 or > 100)
		{
			ErrorMessage = "Enter a title between 2 and 100 characters.";
			return;
		}

		try
		{
			IsBusy = true;
			ErrorMessage = string.Empty;
			await api.CreateAsync(new UpsertAssignmentRequest
			{
				Title = Title.Trim(),
				DueDate = DueDate.Date,
				IsCompleted = IsCompleted
			});
			await Shell.Current.GoToAsync("..");
		}
		catch (Exception exception) when (exception is HttpRequestException or TaskCanceledException)
		{
			ErrorMessage = "The assignment could not be saved. Check the API and tunnel.";
		}
		finally
		{
			IsBusy = false;
		}
	}

	[RelayCommand]
	private static Task CancelAsync() => Shell.Current.GoToAsync("..");
}
```

Create `src/CourseAssignments.App/ViewModels/EditAssignmentViewModel.cs`:

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using CourseAssignments.App.Services;
using CourseAssignments.Contracts.Models;

namespace CourseAssignments.App.ViewModels;

public partial class EditAssignmentViewModel(AssignmentApiService api)
	: ObservableObject, IQueryAttributable
{
	[ObservableProperty] private int assignmentId;
	[ObservableProperty] private string title = string.Empty;
	[ObservableProperty] private DateTime dueDate = DateTime.Today;
	[ObservableProperty] private bool isCompleted;
	[ObservableProperty] private bool isBusy;
	[ObservableProperty] private string errorMessage = string.Empty;

	public bool HasError => !string.IsNullOrWhiteSpace(ErrorMessage);
	partial void OnErrorMessageChanged(string value) => OnPropertyChanged(nameof(HasError));

	public void ApplyQueryAttributes(IDictionary<string, object> query)
	{
		if (query.TryGetValue("id", out var value)
			&& int.TryParse(value?.ToString(), out var id))
			AssignmentId = id;
	}

	public async Task LoadAsync()
	{
		if (AssignmentId <= 0 || IsBusy)
			return;

		try
		{
			IsBusy = true;
			ErrorMessage = string.Empty;
			var item = await api.GetByIdAsync(AssignmentId);
			if (item is null)
			{
				ErrorMessage = "This assignment no longer exists.";
				return;
			}

			Title = item.Title;
			DueDate = item.DueDate;
			IsCompleted = item.IsCompleted;
		}
		catch (Exception exception) when (exception is HttpRequestException or TaskCanceledException)
		{
			ErrorMessage = "The assignment could not be retrieved.";
		}
		finally
		{
			IsBusy = false;
		}
	}

	[RelayCommand]
	private async Task SaveAsync()
	{
		if (IsBusy)
			return;

		if (string.IsNullOrWhiteSpace(Title) || Title.Trim().Length is < 2 or > 100)
		{
			ErrorMessage = "Enter a title between 2 and 100 characters.";
			return;
		}

		try
		{
			IsBusy = true;
			ErrorMessage = string.Empty;
			await api.UpdateAsync(AssignmentId, new UpsertAssignmentRequest
			{
				Title = Title.Trim(),
				DueDate = DueDate.Date,
				IsCompleted = IsCompleted
			});
			await Shell.Current.GoToAsync("..");
		}
		catch (Exception exception) when (exception is HttpRequestException or TaskCanceledException)
		{
			ErrorMessage = "The assignment could not be updated.";
		}
		finally
		{
			IsBusy = false;
		}
	}

	[RelayCommand]
	private static Task CancelAsync() => Shell.Current.GoToAsync("..");
}
```

The edit page receives only an ID and retrieves the current record from `GET /api/assignments/{id}`. This avoids editing a stale copy from the list.

## 10. Register services and pages

Replace `src/CourseAssignments.App/MauiProgram.cs`:

```csharp
using CourseAssignments.App.Models;
using CourseAssignments.App.Services;
using CourseAssignments.App.ViewModels;
using CourseAssignments.App.Views;

#if MAUI_DEVFLOW
using Microsoft.Maui.DevFlow.Agent;
#endif

namespace CourseAssignments.App;

public static class MauiProgram
{
	public static MauiApp CreateMauiApp()
	{
		var builder = MauiApp.CreateBuilder();
		builder.UseMauiApp<App>()
			.ConfigureFonts(fonts =>
			{
				fonts.AddFont("OpenSans-Regular.ttf", "OpenSansRegular");
				fonts.AddFont("OpenSans-Semibold.ttf", "OpenSansSemibold");
			});

#if MAUI_DEVFLOW
		builder.AddMauiDevFlowAgent();
#endif

		builder.Services.AddHttpClient<AssignmentApiService>(client =>
		{
			client.BaseAddress = new Uri(ApiSettings.BaseUrl);
			client.Timeout = TimeSpan.FromSeconds(15);
		});

		builder.Services.AddSingleton<AppShell>();
		builder.Services.AddSingleton<AssignmentsViewModel>();
		builder.Services.AddTransient<AddAssignmentViewModel>();
		builder.Services.AddTransient<EditAssignmentViewModel>();
		builder.Services.AddSingleton<AssignmentsPage>();
		builder.Services.AddTransient<AddAssignmentPage>();
		builder.Services.AddTransient<EditAssignmentPage>();
		return builder.Build();
	}
}
```

The VS Code MAUI extension supplies a compatible DevFlow package for DevFlow-enabled builds. Do not pin a different agent version unless instructed.

Replace `src/CourseAssignments.App/App.xaml.cs`:

```csharp
namespace CourseAssignments.App;

public partial class App : Application
{
	private readonly AppShell appShell;

	public App(AppShell appShell)
	{
		InitializeComponent();
		this.appShell = appShell;
	}

	protected override Window CreateWindow(IActivationState? activationState) => new(appShell);
}
```

Replace `src/CourseAssignments.App/AppShell.xaml`:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<Shell
	x:Class="CourseAssignments.App.AppShell"
	xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
	xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
	xmlns:views="clr-namespace:CourseAssignments.App.Views"
	FlyoutBehavior="Disabled"
	Shell.BackgroundColor="#006D77"
	Shell.ForegroundColor="White"
	Title="Course Assignments">
	<ShellContent
		Title="Assignments"
		Route="assignments"
		ContentTemplate="{DataTemplate views:AssignmentsPage}" />
</Shell>
```

Replace `src/CourseAssignments.App/AppShell.xaml.cs`:

```csharp
using CourseAssignments.App.Views;

namespace CourseAssignments.App;

public partial class AppShell : Shell
{
	public AppShell()
	{
		InitializeComponent();
		Routing.RegisterRoute(nameof(AddAssignmentPage), typeof(AddAssignmentPage));
		Routing.RegisterRoute(nameof(EditAssignmentPage), typeof(EditAssignmentPage));
	}
}
```

Shell is the only navigation container. Do not wrap it in `NavigationPage`, `TabbedPage`, or `FlyoutPage`.

## 11. Build the landing list page

Create `src/CourseAssignments.App/Views/AssignmentsPage.xaml`:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage
	x:Class="CourseAssignments.App.Views.AssignmentsPage"
	xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
	xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
	xmlns:contracts="clr-namespace:CourseAssignments.Contracts.Models;assembly=CourseAssignments.Contracts"
	xmlns:vm="clr-namespace:CourseAssignments.App.ViewModels"
	x:DataType="vm:AssignmentsViewModel"
	Background="#F7F8FA"
	Title="Assignments">

	<Grid Padding="16" RowDefinitions="Auto,Auto,*,Auto" RowSpacing="12">
		<Grid ColumnDefinitions="*,Auto" ColumnSpacing="12">
			<VerticalStackLayout Spacing="2">
				<Label FontAttributes="Bold" FontSize="24"
					   Text="Course assignments" TextColor="#17202A" />
				<Label Text="Data from the remote API" TextColor="#52606D" />
			</VerticalStackLayout>
			<Button Grid.Column="1" AutomationId="AddAssignmentButton"
					Background="#006D77" Command="{Binding AddCommand}"
					Text="Add" TextColor="White" />
		</Grid>

		<Label Grid.Row="1" AutomationId="ListErrorLabel"
			   IsVisible="{Binding HasError}" Text="{Binding ErrorMessage}"
			   TextColor="#B42318" />

		<RefreshView Grid.Row="2" Command="{Binding LoadCommand}"
					 IsRefreshing="{Binding IsBusy}">
			<CollectionView AutomationId="AssignmentsCollection"
							ItemsSource="{Binding Assignments}"
							SelectionMode="None">
				<CollectionView.EmptyView>
					<VerticalStackLayout Padding="24" Spacing="8"
										 HorizontalOptions="Center"
										 VerticalOptions="Center">
						<Label FontAttributes="Bold" Text="No assignments yet"
							   TextColor="#17202A" />
						<Label Text="Use Add to create the first assignment."
							   TextColor="#52606D" />
					</VerticalStackLayout>
				</CollectionView.EmptyView>
				<CollectionView.ItemTemplate>
					<DataTemplate x:DataType="contracts:AssignmentDto">
						<Border Margin="0,0,0,10" Padding="14"
								Background="White" Stroke="#CBD5E1"
								StrokeShape="RoundRectangle 8">
							<Grid RowDefinitions="Auto,Auto,Auto"
								  ColumnDefinitions="*,Auto"
								  ColumnSpacing="12" RowSpacing="8">
								<Label FontAttributes="Bold" FontSize="18"
									   Text="{Binding Title}" TextColor="#17202A" />
								<CheckBox Grid.Column="1" IsEnabled="False"
										  IsChecked="{Binding IsCompleted, Mode=OneWay}" />
								<Label Grid.Row="1" Grid.ColumnSpan="2"
									   Text="{Binding DueDate, StringFormat='Due {0:MMM d, yyyy}'}"
									   TextColor="#52606D" />
								<HorizontalStackLayout Grid.Row="2" Grid.ColumnSpan="2" Spacing="8">
									<Button Background="#006D77" Text="Edit" TextColor="White"
											Command="{Binding Source={RelativeSource AncestorType={x:Type vm:AssignmentsViewModel}}, Path=EditCommand}"
											CommandParameter="{Binding .}" />
									<Button Background="#B42318" Text="Delete" TextColor="White"
											Command="{Binding Source={RelativeSource AncestorType={x:Type vm:AssignmentsViewModel}}, Path=DeleteCommand}"
											CommandParameter="{Binding .}" />
								</HorizontalStackLayout>
							</Grid>
						</Border>
					</DataTemplate>
				</CollectionView.ItemTemplate>
			</CollectionView>
		</RefreshView>

		<ActivityIndicator Grid.Row="3" Color="#006D77"
						   IsRunning="{Binding IsBusy}"
						   IsVisible="{Binding IsBusy}" />
	</Grid>
</ContentPage>
```

Use `CollectionView`, not obsolete `ListView`. Its star-sized Grid row lets it scroll and virtualize correctly; never place it inside a vertical stack.

Create `src/CourseAssignments.App/Views/AssignmentsPage.xaml.cs`:

```csharp
using CourseAssignments.App.ViewModels;

namespace CourseAssignments.App.Views;

public partial class AssignmentsPage : ContentPage
{
	private readonly AssignmentsViewModel viewModel;

	public AssignmentsPage(AssignmentsViewModel viewModel)
	{
		InitializeComponent();
		BindingContext = viewModel;
		this.viewModel = viewModel;
	}

	protected override async void OnAppearing()
	{
		base.OnAppearing();
		await viewModel.LoadAsync();
	}
}
```

The lifecycle override delegates to the view model. It contains no HTTP or CRUD business logic.

## 12. Build the add page

Create `src/CourseAssignments.App/Views/AddAssignmentPage.xaml`:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage
	x:Class="CourseAssignments.App.Views.AddAssignmentPage"
	xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
	xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
	xmlns:vm="clr-namespace:CourseAssignments.App.ViewModels"
	x:DataType="vm:AddAssignmentViewModel"
	Background="#F7F8FA"
	Title="Add assignment">
	<Grid>
		<ScrollView>
			<VerticalStackLayout Padding="20" Spacing="14">
				<Label FontAttributes="Bold" FontSize="24"
					   Text="New assignment" TextColor="#17202A" />
				<Label Text="Title" TextColor="#17202A" />
				<Entry AutomationId="AddTitleEntry" Background="White"
					   Placeholder="Example: Finish API lab"
					   PlaceholderColor="#52606D"
					   Text="{Binding Title, Mode=TwoWay}" TextColor="#17202A" />
				<Label Text="Due date" TextColor="#17202A" />
				<DatePicker AutomationId="AddDueDatePicker" Background="White"
							Date="{Binding DueDate, Mode=TwoWay}" TextColor="#17202A" />
				<HorizontalStackLayout Spacing="8">
					<CheckBox AutomationId="AddCompletedCheckBox"
							  IsChecked="{Binding IsCompleted, Mode=TwoWay}" />
					<Label Text="Completed" TextColor="#17202A"
						   VerticalOptions="Center" />
				</HorizontalStackLayout>
				<Label AutomationId="AddErrorLabel" IsVisible="{Binding HasError}"
					   Text="{Binding ErrorMessage}" TextColor="#B42318" />
				<Grid ColumnDefinitions="*,*" ColumnSpacing="10">
					<Button AutomationId="SaveNewAssignmentButton"
							Background="#006D77" Command="{Binding SaveCommand}"
							Text="Save" TextColor="White" />
					<Button Grid.Column="1" Background="#E2E8F0"
							Command="{Binding CancelCommand}" Text="Cancel"
							TextColor="#17202A" />
				</Grid>
				<ActivityIndicator Color="#006D77" IsRunning="{Binding IsBusy}"
								   IsVisible="{Binding IsBusy}" />
			</VerticalStackLayout>
		</ScrollView>
	</Grid>
</ContentPage>
```

Create `src/CourseAssignments.App/Views/AddAssignmentPage.xaml.cs`:

```csharp
using CourseAssignments.App.ViewModels;

namespace CourseAssignments.App.Views;

public partial class AddAssignmentPage : ContentPage
{
	public AddAssignmentPage(AddAssignmentViewModel viewModel)
	{
		InitializeComponent();
		BindingContext = viewModel;
	}
}
```

## 13. Build the edit page

Create `src/CourseAssignments.App/Views/EditAssignmentPage.xaml`:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage
	x:Class="CourseAssignments.App.Views.EditAssignmentPage"
	xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
	xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
	xmlns:vm="clr-namespace:CourseAssignments.App.ViewModels"
	x:DataType="vm:EditAssignmentViewModel"
	Background="#F7F8FA"
	Title="Edit assignment">
	<Grid>
		<ScrollView>
			<VerticalStackLayout Padding="20" Spacing="14">
				<Label FontAttributes="Bold" FontSize="24"
					   Text="Edit assignment" TextColor="#17202A" />
				<Label Text="{Binding AssignmentId, StringFormat='Assignment #{0}'}"
					   TextColor="#52606D" />
				<Label Text="Title" TextColor="#17202A" />
				<Entry AutomationId="EditTitleEntry" Background="White"
					   Placeholder="Assignment title" PlaceholderColor="#52606D"
					   Text="{Binding Title, Mode=TwoWay}" TextColor="#17202A" />
				<Label Text="Due date" TextColor="#17202A" />
				<DatePicker AutomationId="EditDueDatePicker" Background="White"
							Date="{Binding DueDate, Mode=TwoWay}" TextColor="#17202A" />
				<HorizontalStackLayout Spacing="8">
					<CheckBox AutomationId="EditCompletedCheckBox"
							  IsChecked="{Binding IsCompleted, Mode=TwoWay}" />
					<Label Text="Completed" TextColor="#17202A"
						   VerticalOptions="Center" />
				</HorizontalStackLayout>
				<Label AutomationId="EditErrorLabel" IsVisible="{Binding HasError}"
					   Text="{Binding ErrorMessage}" TextColor="#B42318" />
				<Grid ColumnDefinitions="*,*" ColumnSpacing="10">
					<Button AutomationId="SaveEditedAssignmentButton"
							Background="#006D77" Command="{Binding SaveCommand}"
							Text="Update" TextColor="White" />
					<Button Grid.Column="1" Background="#E2E8F0"
							Command="{Binding CancelCommand}" Text="Cancel"
							TextColor="#17202A" />
				</Grid>
				<ActivityIndicator Color="#006D77" IsRunning="{Binding IsBusy}"
								   IsVisible="{Binding IsBusy}" />
			</VerticalStackLayout>
		</ScrollView>
	</Grid>
</ContentPage>
```

Create `src/CourseAssignments.App/Views/EditAssignmentPage.xaml.cs`:

```csharp
using CourseAssignments.App.ViewModels;

namespace CourseAssignments.App.Views;

public partial class EditAssignmentPage : ContentPage
{
	private readonly EditAssignmentViewModel viewModel;

	public EditAssignmentPage(EditAssignmentViewModel viewModel)
	{
		InitializeComponent();
		BindingContext = viewModel;
		this.viewModel = viewModel;
	}

	protected override async void OnAppearing()
	{
		base.OnAppearing();
		await viewModel.LoadAsync();
	}
}
```

Delete the default generated `Class1.cs`, WeatherForecast files, and the MAUI `MainPage.xaml` files from the existing projects.

## 14. Launch and verify the app

Before launching, confirm all three network layers in order:

1. The API is running on `http://localhost:5050`.
2. The Dev Tunnel is running and its HTTPS URL returns `200 OK`.
3. `ApiSettings.BaseUrl` contains that exact HTTPS URL with a trailing slash.

Start the existing emulator with the MAUI CLI if it is not already running, select it in the VS Code MAUI target selector, and select `CourseAssignments.App.csproj` as the startup project. Launch with the .NET MAUI debugger. The launch builds the selected target, so do not run a separate build first.

If launch fails with a compiler error, diagnose it with:

```console
dotnet build src/CourseAssignments.App/CourseAssignments.App.csproj -f net11.0-android -p:MauiDevFlowEnabled=true
```

Fix the first relevant error and launch again.

### Required manual workflow

1. Confirm the app opens on the assignment list.
2. Pull down to retrieve the current list.
3. Tap **Add**, enter valid values, and save.
4. Confirm the new database record appears on the list.
5. Tap **Edit** and confirm the page retrieves that record by ID.
6. Change its title, date, or completed state and tap **Update**.
7. Confirm the updated values appear on the refreshed list.
8. Tap **Delete** and confirm the record disappears.
9. Restart only the mobile app and confirm undeleted records remain in SQLite.
10. Stop the API and refresh. Confirm the app shows a useful error instead of crashing.
11. Restart the API and tunnel. If the URL changed, update `ApiSettings.BaseUrl`, redeploy, and verify recovery.

Use Hot Reload first after XAML or C# edits while a debug session is active. Rebuild only when Hot Reload cannot apply the change.

## 15. Verify with DevFlow

From the MAUI project directory, initialize the GitHub tooling if needed:

```console
maui devflow init --target github
```

With the app running:

1. Use `maui_tree` to prove `AssignmentsCollection` is visible with non-zero bounds.
2. Use `maui_screenshot` to capture the populated landing page.
3. Navigate to Add and query `AddTitleEntry` with `maui_get_property`.
4. Record its runtime `BackgroundColor`, `TextColor`, and `PlaceholderColor`.
5. Query `AddDueDatePicker` for runtime `BackgroundColor` and `TextColor`.
6. Use `maui_fill` and `maui_tap` to create a record.
7. Use `maui_assert` to prove the new title appears.
8. Capture the edit page and the list after deletion.

Native Android controls can override colors declared in XAML. Source code alone cannot prove accessible contrast. Inspect the actual runtime values and screenshots. Dark text such as `#17202A` needs a light background; white text needs a dark background.

Include both statements with real values:

```text
Verified: AssignmentsCollection found at bounds (x,y,width,height), visible, with the expected records.
Accessibility: AddTitleEntry TextColor=#...... on Background=#......, PlaceholderColor=#......; contrast is sufficient.
```

If no DevFlow agent connects, wait once for startup. If it still fails, confirm the conditional `AddMauiDevFlowAgent()` registration, inspect MAUI debug output, and verify the app launched with DevFlow enabled.

## Expected error behavior

| Situation | Required result |
| --- | --- |
| Empty database | CollectionView empty state appears |
| Blank or one-character title | App shows validation and sends no request |
| Invalid API body | API returns `400 Bad Request` |
| Missing ID | API returns `404 Not Found`; edit shows a useful message |
| API/tunnel unavailable | App shows a connection error and does not crash |
| Successful create | API returns `201 Created`; app returns to list |
| Successful update/delete | API returns `204 No Content` |

Do not silently catch all exceptions. These view models catch expected HTTP and timeout failures while leaving programming errors visible to the debugger.

## Networking troubleshooting map

Diagnose from the API outward instead of changing several settings at once.

| Symptom | Likely layer | Check |
| --- | --- | --- |
| Local URL fails | API | Process, port 5050, migration error |
| Local works; tunnel fails | Tunnel | Host process, port, protocol, login |
| Tunnel works on desktop; Android fails | App/device | URL, internet permission, emulator connectivity |
| Android reports connection refused to localhost | Address | `localhost` points to Android, not the computer |
| Android reports cleartext blocked | Protocol | Use tunnel HTTPS; do not switch to local HTTP |
| Android reports certificate error | TLS trust | Use the tunnel certificate, not the local development certificate |
| Requests suddenly stop after restart | Temporary URL | Copy the new URL into `ApiSettings` and redeploy |
| Physical device cannot use `10.0.2.2` | Network topology | That alias is emulator-specific; use the tunnel |

### Tunnel returns 502

- Verify `http://localhost:5050/api/assignments` first.
- Confirm the tunnel forwards port 5050 with protocol `http`.
- Check that neither terminal was closed.
- For more output, run `devtunnel --verbose host -p 5050 --protocol http --allow-anonymous`.

### API works in the desktop browser but not Android

- Verify the app uses the tunnel URL, not `localhost`, `127.0.0.1`, or `10.0.2.2`.
- Verify the URL starts with `https://` and ends with `/`.
- Verify the Android manifest contains the `INTERNET` permission.
- Check MAUI debug output, tunnel output, and API output for the same request.
- Confirm the emulator itself has internet access.

### EF Core fails

- Confirm every EF Core package and `dotnet-ef` uses major version 11.
- Read the first migration or startup error.
- Do not delete migrations or the database unless instructed to reset the lab.

### Shell cannot create a page

- Confirm every page and view model is registered in DI.
- Confirm add/edit routes are registered in `AppShell`.
- Confirm each page constructor receives the matching view model.

### A binding does not update

- Confirm the page and item template have the correct `x:DataType`.
- Confirm each view model is `partial` and inherits `ObservableObject`.
- Bind to generated public names such as `Title` and `SaveCommand`, not private fields.

## Submission requirements

Submit:

1. The complete solution with Contracts, API, and MAUI projects.
2. The generated EF migration. Do not submit `bin` or `obj` folders.
3. `requests.http` with recorded CRUD status results.
4. Screenshots of the populated list, add page, retrieved edit record, updated list, list after deletion, and network-error state.
5. DevFlow evidence with CollectionView bounds/visibility, runtime Entry and DatePicker colors, one successful assertion, and the required verification statements.

Do not submit credentials, tunnel tokens, or private data. Stop the temporary tunnel before finishing.

## Acceptance checklist

- [ ] Uses an installed .NET 11 SDK and compatible MAUI workload
- [ ] Has separate Contracts, API, and MAUI projects
- [ ] Uses an EF Core SQLite migration
- [ ] Keeps the database behind the API
- [ ] Implements GET-all, GET-by-ID, POST, PUT, and DELETE
- [ ] Returns appropriate `200`, `201`, `204`, `400`, and `404` responses
- [ ] Includes a working `requests.http` file
- [ ] Uses a temporary HTTPS Dev Tunnel for Android communication
- [ ] Explains why Android cannot use the host's `localhost`
- [ ] Opens on a `CollectionView` list
- [ ] Uses separate add and edit pages
- [ ] Uses MVVM Toolkit, commands, DI, and compiled bindings
- [ ] Retrieves an individual record before editing
- [ ] Shows loading, empty, validation, and network-error states
- [ ] Persists CRUD changes in the remote SQLite database
- [ ] Includes DevFlow tree, screenshot, interaction, assertion, and runtime color evidence
- [ ] Uses no `ListView`, `TableView`, nested navigation container, or CRUD logic in code-behind

## Grading rubric (100 points)

| Category | Points |
| --- | ---: |
| Solution structure and contracts | 10 |
| EF Core SQLite persistence | 15 |
| Complete API CRUD and status codes | 20 |
| MAUI MVVM, DI, and HTTP service | 15 |
| List, add, edit, and navigation UI | 15 |
| Validation and UI states | 10 |
| Manual API and end-to-end verification | 5 |
| Android networking and Dev Tunnel practices | 5 |
| DevFlow and accessibility evidence | 5 |
| **Total** | **100** |

Full credit requires working end-to-end behavior and convincing evidence. Partial credit applies when a feature is incomplete or weakly verified. A local-only collection that bypasses the Web API does not satisfy CRUD persistence requirements.

## Explain the final data flow

Be prepared to explain:

1. The user taps **Save** in MAUI.
2. The view model validates input.
3. `AssignmentApiService` serializes the request as JSON.
4. HTTPS carries it to the Dev Tunnel.
5. The tunnel forwards it to `http://localhost:5050` on the computer.
6. The controller validates the request and creates an EF entity.
7. EF Core inserts the row into SQLite.
8. The API returns `201 Created` with a DTO.
9. The app returns to the list and retrieves the current records.

If you can demonstrate and explain this path, you understand the central lesson of the assignment.
