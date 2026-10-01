# Assignment: Convert the Local Notes App to LiteDB

## Overview

In this assignment, you will convert the completed .NET 11 MAUI Local Notes app from the SQLite assignment to use LiteDB, an embedded document database.

The SQLite solution must already build, run, and preserve notes after a restart. You will keep that implementation in the project and add a parallel LiteDB implementation with different class and page names. When the conversion is complete, LiteDB will be the active data store and the original SQLite code will remain unchanged and compilable.

The finished app still has one content page and the same create, read, update, and delete behavior. The important change is how each note is represented and stored.

No web server, internet connection, account, API key, or remote database is required after NuGet packages have been restored.

## Learning outcomes

By the end of this assignment, you should be able to:

- explain the difference between a relational database and a document database
- add LiteDB to an existing .NET 11 MAUI project
- map a C# class to a BSON document in a LiteDB collection
- use a LiteDB document identity for create and update decisions
- keep synchronous database work off the MAUI UI thread
- serialize access to a shared embedded database connection
- preserve two data-access implementations side by side
- switch dependency-injection and AppShell wiring to a new implementation
- verify create, read, update, delete, and restart persistence

## Starting point and conversion rules

Begin with the completed `LocalNotes` solution from the SQLite assignment. Verify that it works before changing the active implementation.

The existing solution should contain these SQLite-specific files:

```text
src/LocalNotes.App/
|-- Models/
|   `-- Note.cs
|-- Data/
|   `-- NoteDatabase.cs
|-- ViewModels/
|   `-- MainViewModel.cs
|-- Views/
|   |-- MainPage.xaml
|   `-- MainPage.xaml.cs
|-- App.xaml.cs
|-- AppShell.xaml
|-- AppShell.xaml.cs
|-- GlobalXamlNamespaces.cs
`-- MauiProgram.cs
```

Follow these rules throughout the conversion:

1. Do not edit, rename, exclude, or delete the five SQLite-specific files.
2. Do not remove the `sqlite-net-pcl` package.
3. Create new LiteDB files with `LiteDb` in each type name.
4. Keep the existing SQLite dependency-injection registrations.
5. Modify only the shared package configuration, `MauiProgram.cs`, and `AppShell.xaml`.
6. Keep `App.xaml.cs`, `AppShell.xaml.cs`, and `GlobalXamlNamespaces.cs` unchanged.

These rules leave both implementations compilable. AppShell selects which page is active.

## From SQLite to LiteDB

The SQLite implementation has this path:

```text
AppShell route: //notes
	|
	v
MainPage
	|
	v
MainViewModel
	|
	v
NoteDatabase
	|
	v
sqlite-net-pcl
	|
	v
notes.db3
```

The new LiteDB implementation has a separate path:

```text
AppShell route: //litedb-notes
	|
	v
LiteDbMainPage
	|
	v
LiteDbMainViewModel
	|
	v
LiteDbNoteDatabase
	|
	v
LiteDB notes collection
	|
	v
notes-litedb.db
```

The distinct file names are important. A SQLite database file and a LiteDB database file use different formats and must not overwrite one another.

## SQLite and LiteDB compared

Both products are embedded databases: they run in the app process and store data in a local file. Neither choice is automatically better for every app.

| Concern | SQLite solution | LiteDB solution |
| --- | --- | --- |
| Data organization | tables and rows | collections and BSON documents |
| Mapping package | `sqlite-net-pcl` | `LiteDB` |
| Primary identity | SQLite integer primary key | LiteDB `_id` field |
| Initialization | create a table | get a collection and create an index |
| Query API | `Table<T>()` | `GetCollection<T>()` and `Query()` |
| API style used here | asynchronous wrapper API | synchronous API moved off the UI thread |
| Local file | `notes.db3` | `notes-litedb.db` |

LiteDB maps a C# object to a BSON document. BSON is a binary document representation that supports named fields and values. In this assignment, documents still have a consistent shape because every document is mapped from the same C# class.

> Important: local storage is not cloud storage. LiteDB notes remain on the current device, are not automatically synchronized, and are normally removed when the app is uninstalled or its data is cleared.

## 1. Recheck the existing app

Open a terminal in the completed `LocalNotes` solution folder and record the environment:

```console
dotnet --version
dotnet workload list
maui --version
```

Launch the SQLite app on the target you will use for this assignment. Create a note, stop the debug session, relaunch the app, and confirm that the note returns.

Resolve existing build, deployment, or persistence errors before adding LiteDB. Otherwise, you will not know whether a later failure was introduced by the conversion.

## 2. Add the LiteDB package

Run these commands from the `LocalNotes` solution folder:

```console
dotnet add src/LocalNotes.App/LocalNotes.App.csproj package LiteDB --version 5.0.21
dotnet list src/LocalNotes.App/LocalNotes.App.csproj package
```

Keep the existing `sqlite-net-pcl` and `CommunityToolkit.Mvvm` package references. The resolved list should now contain all three packages.

LiteDB 5 exposes synchronous database operations. Later sections place those operations on a thread-pool thread so file work does not block input, layout, or rendering on the MAUI UI thread.

## Checkpoint A: verify package coexistence

Build the existing app before adding LiteDB code.

Expected result:

- the project restores and builds
- `LiteDB` version `5.0.21` is resolved
- `sqlite-net-pcl` is still resolved
- `CommunityToolkit.Mvvm` is still resolved
- the existing SQLite files have not changed

Do not uninstall SQLite merely because LiteDB is now installed. The purpose of this assignment is to keep both implementations side by side.

## 3. Create the LiteDB document model

Create `src/LocalNotes.App/Models/LiteDbNote.cs`:

```csharp
using LiteDB;

namespace LocalNotes.App.Models;

public sealed class LiteDbNote
{
	[BsonId]
	public int NoteId { get; set; }

	public string Text { get; set; } = string.Empty;

	public DateTime CreatedAtUtc { get; set; }
}
```

Do not add LiteDB attributes to `Models/Note.cs`.

### How the model becomes a document

`LiteDbNote` describes the C# shape that LiteDB maps to a BSON document.

| C# member | LiteDB purpose |
| --- | --- |
| `LiteDbNote` | document type |
| `NoteId` | `_id` field and document identity |
| `Text` | note content |
| `CreatedAtUtc` | creation timestamp stored in UTC |

`[BsonId]` maps `NoteId` to LiteDB's required `_id` field. A new `LiteDbNote` starts with `NoteId` equal to `0`. LiteDB treats that integer as empty during insertion, generates the next integer identity, and writes it back to the object.

LiteDB does not use the SQLite `[PrimaryKey]`, `[AutoIncrement]`, or `[MaxLength]` mapping attributes. The page's `Entry.MaxLength` and the view model's save condition enforce the 500-character limit for this app.

The model contains data only. It does not open the database, execute commands, or know about pages.

## 4. Create the LiteDB database service

Create `src/LocalNotes.App/Data/LiteDbNoteDatabase.cs`:

```csharp
using LiteDB;
using LocalNotes.App.Models;
using Microsoft.Maui.Storage;

namespace LocalNotes.App.Data;

public sealed class LiteDbNoteDatabase : IDisposable
{
	private readonly LiteDatabase database;
	private readonly ILiteCollection<LiteDbNote> notes;
	private readonly SemaphoreSlim databaseLock = new(1, 1);
	private bool disposed;

	public LiteDbNoteDatabase()
	{
		var databasePath = Path.Combine(
			FileSystem.AppDataDirectory,
			"notes-litedb.db");

		database = new LiteDatabase(databasePath);
		notes = database.GetCollection<LiteDbNote>("notes");
		notes.EnsureIndex(note => note.CreatedAtUtc);
	}

	public async Task<List<LiteDbNote>> GetNotesAsync()
	{
		await databaseLock.WaitAsync();

		try
		{
			ObjectDisposedException.ThrowIf(disposed, this);

			return await Task.Run(() => notes.Query()
				.OrderByDescending(note => note.CreatedAtUtc)
				.ToList());
		}
		finally
		{
			databaseLock.Release();
		}
	}

	public async Task SaveNoteAsync(LiteDbNote note)
	{
		ArgumentNullException.ThrowIfNull(note);
		await databaseLock.WaitAsync();

		try
		{
			ObjectDisposedException.ThrowIf(disposed, this);

			await Task.Run(() =>
			{
				if (note.NoteId == 0)
				{
					notes.Insert(note);
				}
				else if (!notes.Update(note))
				{
					throw new InvalidOperationException(
						$"Note {note.NoteId} no longer exists.");
				}
			});
		}
		finally
		{
			databaseLock.Release();
		}
	}

	public async Task DeleteNoteAsync(LiteDbNote note)
	{
		ArgumentNullException.ThrowIfNull(note);
		await databaseLock.WaitAsync();

		try
		{
			ObjectDisposedException.ThrowIf(disposed, this);

			await Task.Run(() =>
			{
				if (!notes.Delete(note.NoteId))
				{
					throw new InvalidOperationException(
						$"Note {note.NoteId} no longer exists.");
				}
			});
		}
		finally
		{
			databaseLock.Release();
		}
	}

	public void Dispose()
	{
		if (disposed)
			return;

		database.Dispose();
		databaseLock.Dispose();
		disposed = true;
		GC.SuppressFinalize(this);
	}
}
```

### Why the database path matters

`FileSystem.AppDataDirectory` returns the app's private, writable data directory on the current platform. Do not put a writable database in `Resources/Raw`, and do not hard-code a Windows, Android, iOS, or macOS path.

`notes-litedb.db` is deliberately different from the existing `notes.db3`. Never ask LiteDB to open the SQLite file.

### How collection initialization works

`GetCollection<LiteDbNote>("notes")` returns the typed collection interface. LiteDB creates the collection in the file when the first write or index is created.

`EnsureIndex` is safe to call when the index already exists. The `CreatedAtUtc` index supports the query used to display newest notes first. LiteDB does not require `CreateTableAsync` because it does not create relational tables.

### Why the service uses `Task.Run`

LiteDB 5 database methods are synchronous. Calling a file operation directly from a command running on the MAUI UI thread can make the interface unresponsive.

Each service method awaits `Task.Run` so the synchronous operation runs on a thread-pool thread. This keeps the UI responsive, but it does not turn LiteDB into a database with native asynchronous I/O.

### Why access is serialized

The app registers one `LiteDbNoteDatabase` for the process lifetime. `SemaphoreSlim` allows only one service operation at a time and prevents two commands from using or changing the shared connection simultaneously.

The semaphore is released in `finally`, including when LiteDB throws an exception. Exceptions are not hidden or converted into success values; they propagate to the view model, which reports a readable message.

### Why the service is disposable

`LiteDatabase` owns the open data file and must be disposed. The .NET dependency-injection container disposes a singleton that it created when its service provider shuts down. The guard makes repeated `Dispose` calls harmless.

Do not create a new `LiteDatabase` inside every method, and do not wrap the injected singleton service in a page-level `using` statement.

### How CRUD maps to LiteDB

| CRUD operation | Service code |
| --- | --- |
| Create | `notes.Insert(note)` |
| Read | `notes.Query().OrderByDescending(...).ToList()` |
| Update | `notes.Update(note)` |
| Delete | `notes.Delete(note.NoteId)` |

`SaveNoteAsync` chooses insert or update by inspecting the document identity. `NoteId == 0` means the document has not been inserted. A nonzero value identifies an existing document.

An update or delete that returns `false` means the selected document no longer exists. The service throws an `InvalidOperationException` instead of reporting a successful operation that did not happen.

## 5. Create the LiteDB view model

Create `src/LocalNotes.App/ViewModels/LiteDbMainViewModel.cs`:

```csharp
using System.Collections.ObjectModel;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using LocalNotes.App.Data;
using LocalNotes.App.Models;

namespace LocalNotes.App.ViewModels;

public partial class LiteDbMainViewModel : ObservableObject
{
	private const int MaximumNoteLength = 500;
	private readonly LiteDbNoteDatabase database;

	public ObservableCollection<LiteDbNote> Notes { get; } = [];

	[ObservableProperty]
	[NotifyCanExecuteChangedFor(nameof(SaveCommand))]
	private string draftText = string.Empty;

	[ObservableProperty]
	[NotifyCanExecuteChangedFor(nameof(DeleteCommand))]
	private LiteDbNote? selectedNote;

	[ObservableProperty]
	[NotifyCanExecuteChangedFor(nameof(SaveCommand))]
	[NotifyCanExecuteChangedFor(nameof(DeleteCommand))]
	private bool isBusy;

	[ObservableProperty]
	private string statusMessage = "Loading LiteDB notes...";

	public LiteDbMainViewModel(LiteDbNoteDatabase database)
	{
		this.database = database;
	}

	partial void OnSelectedNoteChanged(LiteDbNote? value)
	{
		if (value is null)
			return;

		DraftText = value.Text;
		StatusMessage = $"Editing note {value.NoteId}.";
	}

	private bool CanSave()
	{
		var length = DraftText.Trim().Length;
		return !IsBusy && length is > 0 and <= MaximumNoteLength;
	}

	private bool CanDelete() => !IsBusy && SelectedNote is not null;

	[RelayCommand]
	private async Task LoadAsync()
	{
		if (IsBusy)
			return;

		try
		{
			IsBusy = true;
			await ReloadNotesAsync();
			StatusMessage = Notes.Count == 0
				? "No LiteDB notes yet. Add the first note."
				: $"Loaded {Notes.Count} note(s) from LiteDB.";
		}
		catch (Exception ex)
		{
			StatusMessage = $"Could not load LiteDB notes: {ex.Message}";
		}
		finally
		{
			IsBusy = false;
		}
	}

	[RelayCommand(CanExecute = nameof(CanSave))]
	private async Task SaveAsync()
	{
		try
		{
			IsBusy = true;

			var note = SelectedNote ?? new LiteDbNote
			{
				CreatedAtUtc = DateTime.UtcNow
			};

			note.Text = DraftText.Trim();
			var isNew = note.NoteId == 0;
			await database.SaveNoteAsync(note);
			await ReloadNotesAsync();
			ClearEditor();
			StatusMessage = isNew
				? "Note saved to LiteDB."
				: "Note updated in LiteDB.";
		}
		catch (Exception ex)
		{
			StatusMessage = $"Could not save the note: {ex.Message}";
		}
		finally
		{
			IsBusy = false;
		}
	}

	[RelayCommand(CanExecute = nameof(CanDelete))]
	private async Task DeleteAsync()
	{
		if (SelectedNote is not { } note)
			return;

		try
		{
			IsBusy = true;
			await database.DeleteNoteAsync(note);
			await ReloadNotesAsync();
			ClearEditor();
			StatusMessage = "Note deleted from LiteDB.";
		}
		catch (Exception ex)
		{
			StatusMessage = $"Could not delete the note: {ex.Message}";
		}
		finally
		{
			IsBusy = false;
		}
	}

	[RelayCommand]
	private void NewNote()
	{
		ClearEditor();
		StatusMessage = "Ready for a new LiteDB note.";
	}

	private async Task ReloadNotesAsync()
	{
		var savedNotes = await database.GetNotesAsync();
		Notes.Clear();

		foreach (var note in savedNotes)
			Notes.Add(note);
	}

	private void ClearEditor()
	{
		SelectedNote = null;
		DraftText = string.Empty;
	}
}
```

### How the LiteDB view model works

- `ObservableCollection<LiteDbNote>` notifies the UI when the displayed documents change.
- `[ObservableProperty]` generates bindable properties.
- `[RelayCommand]` generates `LoadCommand`, `SaveCommand`, `DeleteCommand`, and `NewNoteCommand`.
- `CanSave` rejects blank text, text longer than 500 characters, and saves during another operation.
- `CanDelete` requires a selected document and no active operation.
- Selecting a document copies its text into the editor and preserves its nonzero identity.
- Reloading after each change ensures the screen reflects the database file.
- Database exceptions become readable status messages.

The LiteDB view model depends only on `LiteDbNoteDatabase` and `LiteDbNote`. It does not reuse the SQLite `NoteDatabase` or `Note` types.

## 6. Create the LiteDB page

The existing `GlobalXamlNamespaces.cs` already exposes the `Views`, `ViewModels`, `Models`, and `Data` namespaces. The new types use those same namespaces, so do not modify that file.

Create `src/LocalNotes.App/Views/LiteDbMainPage.xaml`:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage
	x:Class="LocalNotes.App.Views.LiteDbMainPage"
	x:DataType="LiteDbMainViewModel"
	Title="LiteDB Notes">

	<Grid
		Padding="20"
		RowDefinitions="Auto,Auto,Auto,Auto,*"
		RowSpacing="12">

		<Label
			Grid.Row="0"
			Text="LiteDB Notes"
			FontAttributes="Bold"
			FontSize="28"
			SemanticProperties.HeadingLevel="Level1" />

		<Entry
			Grid.Row="1"
			AutomationId="LiteDbNoteEntry"
			Text="{Binding DraftText, Mode=TwoWay}"
			Placeholder="Type a note"
			MaxLength="500"
			ReturnType="Done" />

		<Grid
			Grid.Row="2"
			ColumnDefinitions="*,*,*"
			ColumnSpacing="8">
			<Button
				Grid.Column="0"
				AutomationId="LiteDbSaveNoteButton"
				Text="Save note"
				Command="{Binding SaveCommand}" />
			<Button
				Grid.Column="1"
				AutomationId="LiteDbNewNoteButton"
				Text="New"
				Command="{Binding NewNoteCommand}" />
			<Button
				Grid.Column="2"
				AutomationId="LiteDbDeleteNoteButton"
				Text="Delete"
				Command="{Binding DeleteCommand}" />
		</Grid>

		<Grid
			Grid.Row="3"
			ColumnDefinitions="*,Auto"
			ColumnSpacing="12">
			<Label
				VerticalOptions="Center"
				Text="{Binding StatusMessage}"
				LineBreakMode="WordWrap" />
			<ActivityIndicator
				Grid.Column="1"
				IsRunning="{Binding IsBusy}"
				IsVisible="{Binding IsBusy}" />
		</Grid>

		<CollectionView
			Grid.Row="4"
			AutomationId="LiteDbNotesList"
			ItemsSource="{Binding Notes}"
			SelectedItem="{Binding SelectedNote, Mode=TwoWay}"
			SelectionMode="Single">
			<CollectionView.EmptyView>
				<Label
					Text="Saved LiteDB notes will appear here."
					HorizontalOptions="Center"
					VerticalOptions="Center" />
			</CollectionView.EmptyView>
			<CollectionView.ItemTemplate>
				<DataTemplate x:DataType="LiteDbNote">
					<Border
						Margin="0,4"
						Padding="12"
						Stroke="{AppThemeBinding Light=#B8B8B8, Dark=#666666}"
						StrokeShape="RoundRectangle 6"
						StrokeThickness="1">
						<Grid RowDefinitions="Auto,Auto" RowSpacing="4">
							<Label
								Text="{Binding Text}"
								FontSize="17"
								LineBreakMode="WordWrap" />
							<Label
								Grid.Row="1"
								Text="{Binding CreatedAtUtc, StringFormat='Saved {0:u}'}"
								FontSize="12"
								Opacity="0.75" />
						</Grid>
					</Border>
				</DataTemplate>
			</CollectionView.ItemTemplate>
		</CollectionView>
	</Grid>
</ContentPage>
```

The outer Grid gives the `CollectionView` a bounded `*` row where it can scroll and virtualize records. Do not place the `CollectionView` inside a `VerticalStackLayout` or `ScrollView`.

The page-level `x:DataType` and item-template `x:DataType` enable compiled bindings for both the view model and each document.

Create `src/LocalNotes.App/Views/LiteDbMainPage.xaml.cs`:

```csharp
using LocalNotes.App.ViewModels;

namespace LocalNotes.App.Views;

public partial class LiteDbMainPage : ContentPage
{
	private readonly LiteDbMainViewModel viewModel;

	public LiteDbMainPage(LiteDbMainViewModel viewModel)
	{
		InitializeComponent();
		BindingContext = viewModel;
		this.viewModel = viewModel;
	}

	protected override async void OnAppearing()
	{
		base.OnAppearing();
		await viewModel.LoadCommand.ExecuteAsync(null);
	}
}
```

The code-behind contains only page setup and lifecycle behavior. Database and editing behavior remains in the view model.

## 7. Register the LiteDB dependencies

Open `MauiProgram.cs`. Its existing namespace imports from the SQLite assignment already include:

```csharp
using LocalNotes.App.Data;
using LocalNotes.App.ViewModels;
using LocalNotes.App.Views;
```

Keep the SQLite registrations and add the three LiteDB registrations before `builder.Build()`:

```csharp
builder.Services.AddSingleton<NoteDatabase>();
builder.Services.AddSingleton<MainViewModel>();
builder.Services.AddSingleton<MainPage>();

builder.Services.AddSingleton<LiteDbNoteDatabase>();
builder.Services.AddSingleton<LiteDbMainViewModel>();
builder.Services.AddSingleton<LiteDbMainPage>();

builder.Services.AddSingleton<AppShell>();
```

Keep the existing conditional DevFlow integration:

```csharp
#if MAUI_DEVFLOW
builder.AddMauiDevFlowAgent();
#endif
```

Do not replace the conditional with `#if DEBUG`, add a permanent `Microsoft.Maui.DevFlow.Agent` package, or define `MAUI_DEVFLOW` manually.

The default dependency-injection container creates singleton services only when they are requested. Keeping the SQLite registrations does not make the SQLite page active and does not require the LiteDB page to use SQLite.

## 8. Switch AppShell to the LiteDB page

Open `AppShell.xaml`. Change only the active Shell content so it uses the LiteDB page and route:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<Shell
	x:Class="LocalNotes.App.AppShell"
	FlyoutBehavior="Disabled"
	Title="LiteDB Notes">

	<ShellContent
		Title="LiteDB Notes"
		Route="litedb-notes"
		ContentTemplate="{DataTemplate LiteDbMainPage}" />
</Shell>
```

The absolute route is `//litedb-notes`. Do not also call `Routing.RegisterRoute` for this page because a route declared in the Shell visual hierarchy is already registered.

Keep these files unchanged:

- `App.xaml.cs`
- `AppShell.xaml.cs`
- `GlobalXamlNamespaces.cs`
- `Models/Note.cs`
- `Data/NoteDatabase.cs`
- `ViewModels/MainViewModel.cs`
- `Views/MainPage.xaml`
- `Views/MainPage.xaml.cs`

If you use Mac Catalyst, keep the existing network-server entitlement required by DevFlow. The database conversion does not change it.

## Checkpoint B: verify side-by-side compilation

Build and launch the app with the .NET MAUI debugger.

Expected result:

- both `sqlite-net-pcl` and `LiteDB` remain installed
- all SQLite-specific and LiteDB-specific files compile
- AppShell opens `LiteDbMainPage`
- the route is `//litedb-notes`
- the title is **LiteDB Notes**
- the empty message says `Saved LiteDB notes will appear here.`
- no duplicate class, duplicate XAML, or duplicate route error occurs

If the SQLite page opens, confirm `AppShell.xaml` uses `LiteDbMainPage`, not `MainPage`.

## Checkpoint C: create and read a document

Verify this sequence:

1. Confirm the **Save note** button is disabled while the entry is blank.
2. Enter `Review LiteDB document operations`.
3. Select **Save note**.
4. Confirm the note appears in the list.
5. Confirm the status says the note was saved to LiteDB.
6. Confirm the editor is cleared after the reload.

If the page fails before opening, inspect the debug output. Common causes are:

- the LiteDB package was not restored
- a new file namespace does not match its folder or XAML `x:Class`
- the LiteDB services were registered after `builder.Build()`
- `LiteDbMainPage` or `LiteDbMainViewModel` was not registered
- AppShell still names the SQLite page
- `x:DataType` names a SQLite type

## Checkpoint D: update and delete

### Update

1. Select `Review LiteDB document operations` in the list.
2. Confirm its text appears in the entry.
3. Change it to `Review and test LiteDB document operations`.
4. Select **Save note**.
5. Confirm the list displays the updated text once.

The nonzero `NoteId` is preserved when the selected object is edited. That causes `SaveNoteAsync` to call `Update` instead of `Insert`.

### Delete

1. Select the updated note.
2. Confirm **Delete** becomes enabled.
3. Select **Delete**.
4. Confirm the note disappears.
5. Confirm the empty message returns.

## Checkpoint E: prove persistence

Saving and immediately displaying a note does not prove file persistence. The value could still exist only in the running process.

Use this test:

1. Save at least two notes.
2. Record their exact text.
3. Stop the debug session completely.
4. Launch the app again on the same target without uninstalling it or clearing its data.
5. Confirm both notes reload from LiteDB.

Hot Reload alone is not a persistence test because the same app process may continue running.

> Android warning: uninstalling the app, selecting **Clear storage**, or switching to a different emulator/device creates a fresh app-data area. An empty LiteDB collection is expected after any of those actions.

## 9. Inspect the LiteDB location

Temporarily add this line in the `LiteDbNoteDatabase` constructor immediately after `databasePath` is created:

```csharp
System.Diagnostics.Debug.WriteLine($"LiteDB database: {databasePath}");
```

Run the app, find the path in the debug output, and record it for the assignment. Then remove the temporary line.

Do not assume a desktop database extension can directly browse a sandboxed Android or iOS file. Copying files from a device requires platform tooling and is not necessary to prove persistence.

## 10. Verify the running UI with DevFlow

From the MAUI project directory, initialize the GitHub DevFlow tooling once if needed:

```console
maui devflow init --target github
```

Launch the app with the .NET MAUI debugger and wait for the DevFlow agent to connect. Then use the tools available in your course environment:

1. Use `maui_wait` to wait for the running agent.
2. Use `maui_list_agents` and confirm the intended app is registered.
3. Use `maui_capabilities` to inspect the connected agent.
4. Use `maui_tree` to find `LiteDbNoteEntry`, `LiteDbSaveNoteButton`, `LiteDbDeleteNoteButton`, and `LiteDbNotesList`.
5. Confirm each element is visible and has nonzero bounds.
6. Use `maui_fill` and `maui_tap` to save a LiteDB note.
7. Use `maui_assert` to confirm `LiteDbNotesList` contains the new document.
8. Use `maui_get_property` to query the rendered `BackgroundColor`, `TextColor`, and `PlaceholderColor` of `LiteDbNoteEntry`.
9. Use `maui_screenshot` and confirm text is readable in the active theme.

Platform-native controls can override colors declared in XAML. Judge contrast from the running app, not only from source. Normal text should target at least a 4.5:1 contrast ratio.

Record evidence in this form:

```text
Verified: LiteDbNoteEntry and LiteDbNotesList are visible with nonzero bounds; a saved LiteDB note appears in the list.
Accessibility: LiteDbNoteEntry TextColor=____ on BackgroundColor=____, PlaceholderColor=____; contrast is sufficient.
```

If no agent connects, wait once for startup. If it still fails, confirm the selected startup project, the conditional `AddMauiDevFlowAgent()` registration, and the MAUI debug output before changing package references.

## 11. Database concepts to explain

Include short answers to these questions with your submission:

1. How is a LiteDB collection/document different from a SQLite table/row?
2. What BSON field does `[BsonId]` create, and how does a new integer identity receive its value?
3. Why is `FileSystem.AppDataDirectory` better than a hard-coded path?
4. Why does this service move LiteDB work to `Task.Run`?
5. Why does the service use `SemaphoreSlim` even though the page has only one editor?
6. How does `SaveNoteAsync` decide whether to insert or update?
7. Why does the service throw when update or delete returns `false`?
8. What is the responsibility of `LiteDbNoteDatabase`?
9. What is the responsibility of `LiteDbMainViewModel`?
10. Why does stopping and relaunching prove more than Hot Reload?
11. What user actions can remove the local LiteDB file?
12. Why must the SQLite and LiteDB implementations use different database file names?

## Troubleshooting guide

### LiteDB namespace or types are not found

Run `dotnet list src/LocalNotes.App/LocalNotes.App.csproj package` and confirm `LiteDB` 5.0.21 restored. Restore and rebuild the selected MAUI target. Do not add unrelated document-database packages.

### LiteDB code reports SQLite attribute or connection errors

Confirm the new files use `LiteDbNote` and `LiteDbNoteDatabase`. Do not copy `using SQLite;`, `[PrimaryKey]`, `[AutoIncrement]`, or `SQLiteAsyncConnection` into the LiteDB files.

### Every save creates a duplicate

Confirm selection binds to `LiteDbNote? SelectedNote`, selection copies the same object's text into the editor, and save does not replace a selected note with a new object. An existing document must keep its nonzero `NoteId`.

### Update or delete says the note no longer exists

The collection returned `false` for that identity. Reload the list and try again with a current document. Do not hide the failure or report success.

### The UI freezes during database work

Confirm each LiteDB operation is inside the awaited `Task.Run` call in `LiteDbNoteDatabase`. Do not call `notes.Insert`, `notes.Update`, `notes.Delete`, or `notes.Query` directly from the page or view model.

### The database is locked or already disposed

Register `LiteDbNoteDatabase` once as a singleton. Do not create extra `LiteDatabase` connections throughout the app, do not put the injected service in a `using` statement, and do not call `Dispose` from a page.

### The list does not update

Confirm the page binds to `ObservableCollection<LiteDbNote>` and `ReloadNotesAsync` runs after save and delete.

### The buttons never enable

Confirm `[NotifyCanExecuteChangedFor]` names the generated `SaveCommand` and `DeleteCommand` properties. Confirm `DraftText` is not blank or longer than 500 characters.

### The database is empty after relaunch

Confirm you used the same emulator/device, did not uninstall the app, and did not clear its storage. Confirm the service uses `FileSystem.AppDataDirectory`, not a cache directory.

### The SQLite page still opens

Confirm `AppShell.xaml` uses `ContentTemplate="{DataTemplate LiteDbMainPage}"` and route `litedb-notes`.

### XAML reports a compiled-binding error

Confirm the page uses `x:DataType="LiteDbMainViewModel"` and the item template uses `x:DataType="LiteDbNote"`. Confirm `GlobalXamlNamespaces.cs` still exposes the existing app namespaces.

## Student deliverables

Submit:

- the complete `LocalNotes` solution containing both implementations
- `LiteDbNote.cs`
- `LiteDbNoteDatabase.cs`
- `LiteDbMainViewModel.cs`
- `LiteDbMainPage.xaml`
- `LiteDbMainPage.xaml.cs`
- a screenshot showing at least two LiteDB notes
- a screenshot showing one selected LiteDB note ready to edit
- a screenshot after a full stop and relaunch with the same notes present
- the exact .NET SDK, MAUI workload, target platform, and resolved NuGet package versions
- the LiteDB path printed during step 9
- the DevFlow visual-tree and accessibility evidence
- answers to the twelve database concept questions

Do not submit generated `bin` or `obj` folders.

## Acceptance checklist

- [ ] The starting SQLite solution built and persisted data before conversion.
- [ ] The project uses .NET 11 and a compatible MAUI 11 workload.
- [ ] `LiteDB` 5.0.21 is installed.
- [ ] `sqlite-net-pcl` remains installed.
- [ ] `CommunityToolkit.Mvvm` remains installed and is used.
- [ ] The five SQLite-specific model, service, view-model, and page files are unchanged.
- [ ] `LiteDbNote.cs` exists and maps `NoteId` with `[BsonId]`.
- [ ] `LiteDbNoteDatabase.cs` exists and owns one `LiteDatabase`.
- [ ] `LiteDbMainViewModel.cs` exists and uses only LiteDB model/service types.
- [ ] `LiteDbMainPage.xaml` and its code-behind exist.
- [ ] LiteDB uses `notes-litedb.db`, not the SQLite file.
- [ ] The database path uses `FileSystem.AppDataDirectory`.
- [ ] The service uses a typed `notes` collection.
- [ ] The timestamp index is ensured.
- [ ] Synchronous LiteDB work runs through awaited `Task.Run` calls.
- [ ] Shared database access is serialized with `SemaphoreSlim`.
- [ ] The database and semaphore are disposed.
- [ ] Insert, query, update, and delete all work.
- [ ] Missing update/delete targets produce readable failures.
- [ ] Database work remains outside page code-behind.
- [ ] Blank or overlength notes cannot be saved.
- [ ] Database exceptions produce readable status messages.
- [ ] Both SQLite and LiteDB dependency registrations remain.
- [ ] `LiteDbMainPage` is declared as the `litedb-notes` Shell route.
- [ ] `AppShell` is still the application window root.
- [ ] Shell is not wrapped in `NavigationPage`, `TabbedPage`, or `FlyoutPage`.
- [ ] The DevFlow integration remains conditional on `MAUI_DEVFLOW`.
- [ ] No permanent or independently pinned DevFlow agent package is present.
- [ ] Mac Catalyst retains the network-server entitlement when that target is used.
- [ ] XAML uses compiled page and item bindings.
- [ ] The list uses `CollectionView` in a bounded Grid row.
- [ ] The `CollectionView` is not inside a stack layout or scroll view.
- [ ] Notes remain after a full stop and relaunch on the same target.
- [ ] The running UI has nonzero bounds and readable input contrast.

## Grading rubric (100 points)

| Category | Points |
| --- | ---: |
| Package setup and preservation of SQLite implementation | 10 |
| LiteDB document model and identity mapping | 15 |
| Database service, threading, serialization, and disposal | 20 |
| MVVM, commands, dependency injection, and Shell wiring | 20 |
| Create, read, update, and delete behavior | 15 |
| Persistence and runtime/DevFlow verification | 15 |
| Code quality and written explanations | 5 |
| **Total** | **100** |
