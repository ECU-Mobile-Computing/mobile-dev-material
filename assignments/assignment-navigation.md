# Assignment: Northbridge University Navigator

## Overview

Build your first multi-page .NET MAUI application: a student companion for the fictional Northbridge University. Use **.NET 11 and .NET MAUI 11**, XAML, compiled bindings, and CommunityToolkit.Mvvm.

Your main application has exactly four bottom tabs on Android and iPhone:

| Tab | Root page | Purpose |
| --- | --- | --- |
| Home | HomePage | University welcome and shortcuts |
| Courses | CoursesPage | Course catalog and course details |
| Campus | CampusPage | Student services, modal help, and external links |
| Profile | ProfilePage | Student information and a guarded editing workflow |

Desktop and tablet Shell chrome can follow platform conventions. Use an Android emulator/device or iPhone simulator/device to demonstrate the bottom-tab requirement; a Windows or Mac Catalyst run alone is not evidence of a mobile bottom-tab layout.

This is a guided assignment, not just a list of APIs. Complete the main build in order, then perform the comparison labs and submit your navigation evidence. All university data is fictional and in memory. No server, database, account, API key, or paid service is required.

### Learning outcomes

By the end, you should be able to:

- Explain the difference between changing tabs, pushing a detail page, presenting a modal, and opening another application.
- Define Shell's visual hierarchy and register routes for pages outside it.
- Navigate with absolute, relative, contextual, backward, and .NET 11 template routes.
- Pass an ID or an object from a parent to a child, and return a result to the parent.
- Explain the lifetime of a page, its view model, and navigation parameters.
- Inspect, unwind, and deliberately reset navigation history.
- Protect unsaved changes, including attempts to leave with the native Back button.
- Recognize non-Shell navigation alternatives without nesting them inside Shell.
- Verify navigation, bindings, page bounds, and input contrast with DevFlow.

### Navigation coverage map

| Technique | Where you will use it |
| --- | --- |
| Declarative tabs and tab selection | Four-tab AppShell |
| Programmatic absolute navigation | Home shortcut to Courses |
| Relative push navigation | Courses to CourseDetails |
| Query-string data | Course ID lookup |
| Single-use object parameters | Course or instructor object passed to a child |
| Returning data on Back | Favorite-course result; saved profile |
| One-level and multi-level Back | Details, instructor, and editing workflows |
| Resetting a tab to its root | Return to the catalog |
| Shell modal navigation | Campus help |
| Guarded navigation and deferrals | Unsaved profile edits |
| Toolbar Back customization | Accessible detail-page Back button |
| Contextual routes | Courses-specific detail alias |
| .NET 11 path-parameter routes | Open a course by path |
| Browser and OS launcher | University website and email composer |
| Shell flyout and top-tab alternatives | Temporary Shell comparison lab |
| NavigationPage push/pop and stack surgery | Separate non-Shell sandbox |
| Direct modal push/pop | Separate non-Shell sandbox |
| TabbedPage and FlyoutPage | Alternative roots in the sandbox |
| Incoming deep links | Design exercise distinguishing OS activation from Shell routes |

## 1. Understand the navigation model

A **page** is one screen. A **route** is an address for a screen. A **navigation stack** is an ordered history within a navigation container: the most recently pushed page is on top.

```text
App window
	AppShell
		TabBar: university
			Home tab: home       -> welcome
			Courses tab: courses -> catalog -> course-details -> instructor-details
			Campus tab: campus   -> services
			Profile tab: profile -> student -> edit-profile

CampusHelpPage is a modal presented above the current Shell content.
```

The arrows after `catalog` and `student` represent pages pushed at runtime, not additional tabs. Registering a detail route does not add a visible tab.

Think of a course journey as `[CoursesPage, CourseDetailsPage, InstructorDetailsPage]`. Back removes the top page. Forward navigation to another CourseDetailsPage adds a page; it does not mean "go back to the existing one."

Each Shell `Tab` owns its own navigation history. Switching tabs is not the same as pushing one tab onto another tab's history. A modal is a separate presentation layer, not a fifth tab. Routes are not browser history, and neither routes nor in-memory objects survive process termination automatically.

> Architectural rule: use Shell as the main application's navigation container. Do not wrap Shell in NavigationPage, or put NavigationPage, TabbedPage, or FlyoutPage inside it. The separate sandbox near the end exists specifically to compare those alternative architectures.

## 2. Prepare your workspace

The SDK, MAUI, and workloads are already installed. You must still select the right SDK, install/configure an editor, choose a target, and create your application.

### 2.1 Check the installed versions

Open a terminal in a directory where you keep coursework. These commands work in PowerShell and a macOS shell:

```console
dotnet --list-sdks
dotnet --version
dotnet workload list
dotnet new maui --help
```

Select an installed **11.0.100 preview 7 or newer .NET 11 SDK**, together with a compatible MAUI 11 workload. Route templates in this assignment require MAUI 11 Preview 7 or newer. As of September 2026, .NET 11 is prerelease software; do not assume an arbitrary earlier preview has every API below. Record your exact SDK and workload versions in your submission.

Create the workspace:

```console
mkdir UniversityNavigator
cd UniversityNavigator
```

Create `global.json` using your actual installed SDK version. Replace `YOUR_INSTALLED_11_SDK_VERSION` before running this command; it is the complete version printed by `dotnet --list-sdks`, without the directory in brackets.

```console
dotnet new globaljson --sdk-version YOUR_INSTALLED_11_SDK_VERSION --roll-forward latestPatch
```

Open `global.json` and add `"allowPrerelease": true` inside its `sdk` object, keeping the generated `version` and `rollForward` entries. Check `dotnet --version` again **inside this directory**. It must resolve to the intended .NET 11 SDK. Do not copy a preview version from another student's computer unless you also have it installed.

### 2.2 Prepare VS Code and a mobile target

1. Install Visual Studio Code from <https://code.visualstudio.com/> if needed.
2. Open Extensions and install **.NET MAUI**, published by Microsoft. Allow its C# dependencies to install. The extension identifier is `ms-dotnettools.dotnet-maui`.
3. Open this workspace with **File > Open Folder**. If the `code` terminal command is installed, `code .` does the same thing.
4. On Windows, use Android for the mobile demonstration. If you do not have a target, install Android Studio, open **Device Manager**, create a phone virtual device using a system image supported by your installed .NET Android workload, and start it. Install the matching Android SDK/JDK components required by that workload. Alternatively, enable developer options and USB debugging on an Android phone and accept its authorization prompt. Match your target's architecture to the installed workload's supported runtime configuration.
5. On macOS, use the same Android option, or install the Xcode version required by your installed .NET iOS workload, accept its license, install an iOS simulator runtime in Xcode settings, and select a simulator. A physical iPhone additionally requires signing/provisioning. Windows does not provide a local iOS simulator.
6. In VS Code, use **.NET MAUI: Select Startup Project**, then select the device in the MAUI status-bar target selector. Use **.NET MAUI: Refresh Devices** if needed. These selections become available after you create the project below.

Installed MAUI workloads do not guarantee that an emulator, simulator image, Xcode, signing identity, or device connection already exists. Resolve a missing target before attempting the mobile verification.

### 2.3 Create the solution, projects, reference, and package

Run from `UniversityNavigator`:

```console
dotnet new sln --name UniversityNavigator --format slnx
dotnet new maui --name UniversityNavigator.App --output src/UniversityNavigator.App --framework net11.0
dotnet new classlib --name UniversityNavigator.Core --output src/UniversityNavigator.Core --framework net11.0
dotnet sln UniversityNavigator.slnx add src/UniversityNavigator.App/UniversityNavigator.App.csproj
dotnet sln UniversityNavigator.slnx add src/UniversityNavigator.Core/UniversityNavigator.Core.csproj
dotnet add src/UniversityNavigator.App/UniversityNavigator.App.csproj reference src/UniversityNavigator.Core/UniversityNavigator.Core.csproj
dotnet add src/UniversityNavigator.App/UniversityNavigator.App.csproj package CommunityToolkit.Mvvm --version 8.4.0
```

The solution organizes projects. The MAUI project owns platform-specific UI and navigation. The plain .NET class library holds university models and sample data, with no dependency on MAUI. The reference lets the app use the library; adding both projects to the solution does **not** create that reference. The MVVM package generates observable properties and commands; Shell itself is included with MAUI and needs no navigation package. Do not install CommunityToolkit.Maui for this assignment.

Keep the MAUI template's platform-specific `TargetFrameworks`, resources, package references, and platform settings. `net11.0` is the template's framework selector; an actual app build targets a platform TFM such as `net11.0-android`. Do not replace the generated app target frameworks with plain `net11.0`, and do not guess a new `Microsoft.Maui.Controls` version independently of your workload.

Delete `Class1.cs` from the Core project using the editor. Keep the generated `MainPage` until you replace AppShell later; it will then be unused and can be deleted together with its code-behind.

### Checkpoint A: the template runs

Select `UniversityNavigator.App.csproj` as the startup project and launch with the MAUI debugger on your selected device. Confirm that the unmodified template appears. Launch builds the selected target automatically; a separate full solution build is unnecessary and can request platforms you cannot build on this host.

If the template does not launch, resolve the environment error before adding navigation. Record the successful target and device. During later edits, try Hot Reload first; new files, references, or route registrations may require a debugger restart if Hot Reload cannot apply them.

## 3. Add the university model and sample data

Using VS Code Explorer, create a `Models` folder and a `Services` folder inside `src/UniversityNavigator.Core`. Create each file below at the indicated location and replace its contents with the supplied code. Files saved beneath an SDK-style project directory are included automatically; you do not need a command to add each C# file to the project.

### 3.1 Models

Create `src/UniversityNavigator.Core/Models/Instructor.cs`:

```csharp
namespace UniversityNavigator.Core.Models;

public sealed record Instructor(string Name, string Office, string Email);
```

Create `src/UniversityNavigator.Core/Models/Course.cs`:

```csharp
namespace UniversityNavigator.Core.Models;

public sealed record Course(
	string Id,
	string Title,
	int Credits,
	string Description,
	Instructor Instructor);
```

Create `src/UniversityNavigator.Core/Models/StudentProfile.cs`:

```csharp
namespace UniversityNavigator.Core.Models;

public sealed record StudentProfile(string StudentId, string Name, string Program);
```

These records are immutable data snapshots. Passing a record as an object parameter passes a reference, not a serialized copy. Immutability prevents a detail page from accidentally changing the parent's data. Later, saving the profile will deliberately return a new record.

### 3.2 Course catalog

Create `src/UniversityNavigator.Core/Services/CourseCatalog.cs`:

```csharp
using UniversityNavigator.Core.Models;

namespace UniversityNavigator.Core.Services;

public sealed class CourseCatalog
{
	public IReadOnlyList<Course> Courses { get; } = new List<Course>
	{
		new("COMP301", "Mobile Application Development", 3,
			"Build accessible cross-platform applications with .NET MAUI.",
			new Instructor("Dr. Maya Chen", "Computing 214", "maya.chen@example.edu")),
		new("HIST210", "Cities & Communities", 3,
			"Explore how universities and their surrounding communities develop.",
			new Instructor("Dr. Sam Rivera", "Humanities 108", "sam.rivera@example.edu")),
		new("BIOL120", "Introduction to Ecology", 4,
			"Study ecosystems through lectures and campus field observations.",
			new Instructor("Dr. Noor Ahmed", "Science 302", "noor.ahmed@example.edu"))
	}.AsReadOnly();

	public Course? Find(string id) =>
		Courses.FirstOrDefault(course =>
			string.Equals(course.Id, id, StringComparison.OrdinalIgnoreCase));
}
```

The catalog is a local stand-in for a repository or web API. Looking up a course by ID demonstrates a navigation pattern that will still work when the data source changes. An unknown ID must produce a recoverable screen, not a crash.

## 4. Add the view models

Create `ViewModels` inside `src/UniversityNavigator.App`. All files in this section go in that folder. Do not put them beside the solution or in the Core project: they use MAUI's navigation APIs.

The example view models call `Shell.Current` directly to make navigation visible in this introductory assignment. In a larger application, an injected navigation service can hide Shell behind an interface and make unit testing easier. Do not put page navigation in the university models.

CommunityToolkit generates `BrowseCoursesCommand` from `BrowseCoursesAsync`, and `SelectedCourse` from the field `selectedCourse`. Classes with generated members must be `partial`. Asynchronous relay commands disallow concurrent execution of that same command by default, reducing accidental double pushes. Always await navigation; never start several `GoToAsync` operations at once.

### 4.1 HomeViewModel.cs

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace UniversityNavigator.App.ViewModels;

public partial class HomeViewModel : ObservableObject
{
	[RelayCommand]
	private async Task BrowseCoursesAsync() =>
		await Shell.Current.GoToAsync("//university/courses/catalog");
}
```

The leading `//` starts at the Shell hierarchy root. This selects the Courses destination; it does not push CoursesPage on top of HomePage. You will define all three path segments in AppShell.

### 4.2 CoursesViewModel.cs

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using UniversityNavigator.Core.Models;
using UniversityNavigator.Core.Services;

namespace UniversityNavigator.App.ViewModels;

public partial class CoursesViewModel : ObservableObject, IQueryAttributable
{
	private readonly CourseCatalog catalog;

	public IReadOnlyList<Course> Courses => catalog.Courses;

	[ObservableProperty]
	[NotifyCanExecuteChangedFor(nameof(OpenByIdCommand))]
	[NotifyCanExecuteChangedFor(nameof(OpenByObjectCommand))]
	[NotifyCanExecuteChangedFor(nameof(OpenByPathCommand))]
	private Course? selectedCourse;

	[ObservableProperty]
	private string resultMessage = "No favorite course selected.";

	public CoursesViewModel(CourseCatalog catalog)
	{
		this.catalog = catalog;
	}

	private bool HasSelectedCourse() => SelectedCourse is not null;

	[RelayCommand(CanExecute = nameof(HasSelectedCourse))]
	private async Task OpenByIdAsync()
	{
		if (SelectedCourse is not { } course)
			return;

		await Shell.Current.GoToAsync(
			$"course-details?courseId={Uri.EscapeDataString(course.Id)}");
	}

	[RelayCommand(CanExecute = nameof(HasSelectedCourse))]
	private async Task OpenByObjectAsync()
	{
		if (SelectedCourse is not { } course)
			return;

		await Shell.Current.GoToAsync("course-details",
			new ShellNavigationQueryParameters { ["course"] = course });
	}

	[RelayCommand(CanExecute = nameof(HasSelectedCourse))]
	private async Task OpenByPathAsync()
	{
		if (SelectedCourse is not { } course)
			return;

		await Shell.Current.GoToAsync(
			$"//university/courses/catalog/course/{Uri.EscapeDataString(course.Id)}");
	}

	public void ApplyQueryAttributes(IDictionary<string, object> query)
	{
		if (query.TryGetValue("favoriteCourseId", out var value) && value is string encodedId)
		{
			var course = catalog.Find(Uri.UnescapeDataString(encodedId));
			ResultMessage = course is null
				? "The returned course could not be found."
				: $"Favorite course: {course.Id} - {course.Title}";
		}
	}
}
```

The three commands open the same detail page, but deliver data differently. The selected item enables the commands. `IQueryAttributable` receives navigation data on the destination page or its existing BindingContext. Here it also receives a result when a child navigates back to the catalog.

Do not read `CurrentState.Location.Query["courseId"]`: `Uri.Query` is a string, not a parameter dictionary. Let Shell deliver parameters through `ApplyQueryAttributes`.

### 4.3 CourseDetailsViewModel.cs

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using UniversityNavigator.Core.Models;
using UniversityNavigator.Core.Services;

namespace UniversityNavigator.App.ViewModels;

public partial class CourseDetailsViewModel : ObservableObject, IQueryAttributable
{
	private readonly CourseCatalog catalog;

	[ObservableProperty]
	[NotifyCanExecuteChangedFor(nameof(OpenInstructorCommand))]
	[NotifyCanExecuteChangedFor(nameof(ChooseFavoriteCommand))]
	private Course? course;

	[ObservableProperty]
	private string status = "No course supplied. Return to the catalog and select one.";

	public CourseDetailsViewModel(CourseCatalog catalog)
	{
		this.catalog = catalog;
	}

	public void ApplyQueryAttributes(IDictionary<string, object> query)
	{
		if (query.TryGetValue("course", out var courseValue) && courseValue is Course supplied)
		{
			Course = supplied;
		}
		else if (query.TryGetValue("courseId", out var idValue) && idValue is string encodedId)
		{
			Course = catalog.Find(Uri.UnescapeDataString(encodedId));
		}
		else if (query.TryGetValue("pathCourseId", out var pathValue) && pathValue is string pathId)
		{
			Course = catalog.Find(pathId);
		}
		else
		{
			return;
		}

		Status = Course is null ? "Course not found. Return to the catalog." : "";
	}

	private bool HasCourse() => Course is not null;

	[RelayCommand(CanExecute = nameof(HasCourse))]
	private async Task OpenInstructorAsync()
	{
		if (Course is not { } selected)
			return;

		await Shell.Current.GoToAsync("instructor-details",
			new ShellNavigationQueryParameters { ["instructor"] = selected.Instructor });
	}

	[RelayCommand(CanExecute = nameof(HasCourse))]
	private async Task ChooseFavoriteAsync()
	{
		if (Course is not { } selected)
			return;

		await Shell.Current.GoToAsync(
			$"..?favoriteCourseId={Uri.EscapeDataString(selected.Id)}");
	}

	[RelayCommand]
	private async Task BackAsync() => await Shell.Current.GoToAsync("..");

	[RelayCommand]
	private async Task CatalogRootAsync() =>
		await Shell.Current.GoToAsync("//university/courses/catalog");
}
```

Query-string values received by `IQueryAttributable` are not automatically URL-decoded. We encode on the sender and decode once on the receiver. In contrast, **.NET 11 path parameters are already decoded**, so `pathCourseId` deliberately uses a different key and is not decoded again.

The receiver does not erase its existing Course when called without relevant keys. This matters when returning from InstructorDetails: the course page and its view model are still on the stack, but single-use input parameters are not replayed.

### 4.4 InstructorDetailsViewModel.cs

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using UniversityNavigator.Core.Models;

namespace UniversityNavigator.App.ViewModels;

public partial class InstructorDetailsViewModel : ObservableObject, IQueryAttributable
{
	[ObservableProperty]
	private Instructor? instructor;

	public void ApplyQueryAttributes(IDictionary<string, object> query)
	{
		if (query.TryGetValue("instructor", out var value) && value is Instructor supplied)
			Instructor = supplied;
	}

	[RelayCommand]
	private async Task BackAsync() => await Shell.Current.GoToAsync("..");

	[RelayCommand]
	private async Task BackTwoLevelsAsync() => await Shell.Current.GoToAsync("../..");
}
```

The two-level command has a precondition: this page was reached through `CoursesPage -> CourseDetailsPage -> InstructorDetailsPage`. Two `..` operations pop two **pages**, not two strings from the declared TabBar/Tab hierarchy. Do not offer this command from an arbitrary deep-link entry point with a shorter stack; use an absolute catalog route when history is unknown.

### 4.5 CampusViewModel.cs

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using Microsoft.Maui.ApplicationModel;

namespace UniversityNavigator.App.ViewModels;

public partial class CampusViewModel : ObservableObject
{
	[ObservableProperty]
	private string externalStatus = "";

	[RelayCommand]
	private async Task OpenHelpAsync() => await Shell.Current.GoToAsync("campus-help");

	[RelayCommand]
	private async Task OpenWebsiteAsync()
	{
		try
		{
			bool opened = await Browser.Default.OpenAsync(
				new Uri("https://www.example.com/"), BrowserLaunchMode.SystemPreferred);
			ExternalStatus = opened ? "Website opened." : "No browser is available.";
		}
		catch (Exception)
		{
			ExternalStatus = "The website could not be opened on this device.";
		}
	}

	[RelayCommand]
	private async Task EmailServicesAsync()
	{
		try
		{
			bool opened = await Launcher.Default.TryOpenAsync(
				new Uri("mailto:student.services@example.edu"));
			ExternalStatus = opened ? "Email application opened." : "No email application is available.";
		}
		catch (Exception)
		{
			ExternalStatus = "The email application could not be opened on this device.";
		}
	}
}
```

The website uses a reserved example site because the university is fictional. The email address is not a real university inbox: do not send mail. These calls hand off to platform services; they do not push a MAUI detail page. A simulator may have no mail application, which is an expected state handled above. No browser or launcher NuGet package is required.

### 4.6 CampusHelpViewModel.cs

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace UniversityNavigator.App.ViewModels;

public partial class CampusHelpViewModel : ObservableObject
{
	[RelayCommand]
	private async Task CloseAsync() => await Shell.Current.GoToAsync("..");
}
```

The page will opt into modal presentation. Because Shell opens it, Shell also closes it. Do not use `PopModalAsync` to close this route-based example; matching the opening and closing API keeps ownership clear.

### 4.7 ProfileViewModel.cs

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using UniversityNavigator.Core.Models;

namespace UniversityNavigator.App.ViewModels;

public partial class ProfileViewModel : ObservableObject, IQueryAttributable
{
	[ObservableProperty]
	private StudentProfile student = new("N10042", "Alex Morgan", "Computer Science");

	[RelayCommand]
	private async Task EditAsync() =>
		await Shell.Current.GoToAsync("edit-profile",
			new ShellNavigationQueryParameters { ["student"] = Student });

	public void ApplyQueryAttributes(IDictionary<string, object> query)
	{
		if (query.TryGetValue("savedStudent", out var value) && value is StudentProfile saved)
			Student = saved;
	}
}
```

### 4.8 EditProfileViewModel.cs

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using UniversityNavigator.Core.Models;

namespace UniversityNavigator.App.ViewModels;

public partial class EditProfileViewModel : ObservableObject, IQueryAttributable
{
	private StudentProfile? original;

	[ObservableProperty]
	private string name = "";

	[ObservableProperty]
	private bool isDirty;

	[ObservableProperty]
	private string validationMessage = "";

	public void ApplyQueryAttributes(IDictionary<string, object> query)
	{
		if (query.TryGetValue("student", out var value) && value is StudentProfile supplied)
		{
			original = supplied;
			Name = supplied.Name;
			IsDirty = false;
			ValidationMessage = "";
		}
	}

	partial void OnNameChanged(string value)
	{
		IsDirty = original is not null && value != original.Name;
	}

	public void DiscardChanges()
	{
		Name = original?.Name ?? "";
		IsDirty = false;
		ValidationMessage = "";
	}

	[RelayCommand]
	private async Task SaveAsync()
	{
		if (original is null || string.IsNullOrWhiteSpace(Name))
		{
			ValidationMessage = "Enter a student name before saving.";
			return;
		}

		var saved = original with { Name = Name.Trim() };
		IsDirty = false;
		await Shell.Current.GoToAsync("..",
			new ShellNavigationQueryParameters { ["savedStudent"] = saved });
	}

	[RelayCommand]
	private async Task CancelAsync() => await Shell.Current.GoToAsync("..");
}
```

The editor receives a parent snapshot and edits a separate Name property. Save returns a new StudentProfile to the existing parent. Cancel simply requests Back; AppShell will ask before discarding a dirty draft. This is why returning a result and mutating a shared object are not interchangeable.

Do not launch yet: the view models reference routes that you have not registered. Finish the pages and startup wiring next.

## 5. Create the pages

Create `Views` inside `src/UniversityNavigator.App`. For each page below, create both the `.xaml` and `.xaml.cs` files. Use the exact class and namespace spellings. The MAUI SDK includes XAML files automatically as `MauiXaml`; do not add duplicate project entries.

### Set up global XAML namespaces first

Before adding the pages, create `src/UniversityNavigator.App/GlobalXamlNamespaces.cs` using VS Code Explorer. If this file already exists, update its assembly-level mappings instead of adding duplicate mappings. Use the following contents for this application:

```csharp
using Microsoft.Maui.Controls;

[assembly: XmlnsDefinition(
	"http://schemas.microsoft.com/dotnet/2021/maui",
	"UniversityNavigator.App.Views")]

[assembly: XmlnsDefinition(
	"http://schemas.microsoft.com/dotnet/2021/maui",
	"UniversityNavigator.App.ViewModels")]

[assembly: XmlnsDefinition(
	"http://schemas.microsoft.com/dotnet/2021/maui",
	"UniversityNavigator.Core.Models",
	AssemblyName = "UniversityNavigator.Core")]
```

These attributes map our CLR namespaces into MAUI's default XAML namespace. Place them at assembly scope, outside any class or namespace declaration. The filename is an organizational convention, not a special compiler keyword. The SDK automatically compiles this C# file; no additional package or project entry is required.

The Core models live in a separate assembly, so their mapping includes `AssemblyName`. Keep the project reference created in Step 2. The mapping belongs in the MAUI app, not the Core library; Core does not need a MAUI dependency.

.NET 11 supplies the standard MAUI default namespace and XAML language namespace implicitly. Combined with these mappings, our XAML can omit the standard `xmlns` and `xmlns:x` declarations and the custom view/view-model/model declarations. Use `HomeViewModel`, `Course`, and `{DataTemplate HomePage}` without custom prefixes, as shown in the examples below.

Keep `x:` on XAML directives such as `x:Class`, `x:DataType`, `x:Name`, and `x:Key`. Keep the fully qualified `x:Class` value matching its C# code-behind class. Namespace mappings do not set BindingContext, register navigation routes, or register services.

When you add more types to an already mapped namespace, no mapping change is needed. When you introduce a new CLR namespace, add another `XmlnsDefinition` using the same MAUI schema URI and that exact namespace; add `AssemblyName` if it belongs to another referenced assembly. Mappings are not recursive: mapping `UniversityNavigator.App.Views` does not also map `UniversityNavigator.App.Views.Dialogs`. If two mapped namespaces contain types with the same name, retain an explicit XAML namespace prefix where necessary to disambiguate them.

Existing template XAML can retain its explicit standard declarations without affecting these mappings. Remove them when adopting the simplified style, but keep declarations needed for unmapped types. Do not remove the SVG namespace declarations from the image files in Step 6: SVG is not MAUI XAML. These app-level mappings do not automatically configure the separate NavigationLab project.

### 5.1 Add the eight code-behind files

Create `Views/HomePage.xaml.cs` with this complete code:

```csharp
using UniversityNavigator.App.ViewModels;

namespace UniversityNavigator.App.Views;

public partial class HomePage : ContentPage
{
	public HomePage(HomeViewModel viewModel)
	{
		InitializeComponent();
		BindingContext = viewModel;
	}
}
```

Create the other seven code-behind files by using that same code and replacing **all** occurrences of the page and view-model names according to this table. Keep the namespace and using statement unchanged.

| Code-behind file | Page class and constructor | Constructor parameter type |
| --- | --- | --- |
| CoursesPage.xaml.cs | CoursesPage | CoursesViewModel |
| CourseDetailsPage.xaml.cs | CourseDetailsPage | CourseDetailsViewModel |
| InstructorDetailsPage.xaml.cs | InstructorDetailsPage | InstructorDetailsViewModel |
| CampusPage.xaml.cs | CampusPage | CampusViewModel |
| CampusHelpPage.xaml.cs | CampusHelpPage | CampusHelpViewModel |
| ProfilePage.xaml.cs | ProfilePage | ProfileViewModel |
| EditProfilePage.xaml.cs | EditProfilePage | EditProfileViewModel |

For example, CoursesPage's constructor is `public CoursesPage(CoursesViewModel viewModel)`. Dependency injection supplies the view model; you do not manually construct one in XAML. `InitializeComponent` loads the XAML, while `BindingContext` supplies runtime binding data. `x:DataType` in XAML only supplies compile-time type information; it does not create a view model.

**Do we still need `x:DataType` when BindingContext is assigned in code-behind?** Yes, for the compiled XAML bindings required in this assignment. `BindingContext = viewModel` selects the runtime object; it does not tell the XAML compiler that object's type. Keep `x:DataType="CoursesViewModel"` on CoursesPage and `x:DataType="Course"` on its item DataTemplate, whose bindings refer to a different type. Descendants can inherit `x:DataType` from their parent; it need not be repeated on every Label or Button.

The correct syntax is `x:DataType`, not `x.DataType`. It is not universally required for a binding to work: ordinary runtime XAML bindings can work without it, but lose compiled-binding type checking and performance benefits. If you create a binding entirely in C# using a typed expression, such as `label.SetBinding(Label.TextProperty, static (ProfileViewModel viewModel) => viewModel.Student.Name)`, that binding gets its type information from the expression and does not require a XAML `x:DataType`. A string-based C# binding is still a runtime binding; adding `x:DataType` elsewhere does not compile it.

### 5.2 Shared styles

Open the generated `App.xaml`. Keep its existing merged resource dictionaries. Add these **keyed** styles inside `Application.Resources`' existing `ResourceDictionary`, after the merged dictionaries:

```xml
<Style x:Key="UniversityPage" TargetType="ContentPage">
	<Setter Property="Background" Value="{AppThemeBinding Light=#FFFFFF, Dark=#161616}" />
</Style>
<Style x:Key="PageHeading" TargetType="Label">
	<Setter Property="FontSize" Value="26" />
	<Setter Property="FontAttributes" Value="Bold" />
	<Setter Property="TextColor" Value="{AppThemeBinding Light=#183E35, Dark=#B8E4D3}" />
</Style>
<Style x:Key="BodyText" TargetType="Label">
	<Setter Property="FontSize" Value="16" />
	<Setter Property="TextColor" Value="{AppThemeBinding Light=#242424, Dark=#F2F2F2}" />
</Style>
<Style x:Key="ActionButton" TargetType="Button">
	<Setter Property="Background" Value="#185B46" />
	<Setter Property="TextColor" Value="#FFFFFF" />
	<Setter Property="CornerRadius" Value="6" />
	<Setter Property="MinimumHeightRequest" Value="48" />
</Style>
```

Keyed styles avoid collisions with the template's implicit styles. Every page below uses `UniversityPage`. No CollectionView or ScrollView is placed inside a stack layout: a scrollable area must receive a bounded height from its parent Grid.

### 5.3 Views/HomePage.xaml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage x:Class="UniversityNavigator.App.Views.HomePage"
			 x:DataType="HomeViewModel"
			 Style="{StaticResource UniversityPage}" Title="Northbridge">
	<Grid>
		<ScrollView>
			<VerticalStackLayout Padding="24" Spacing="16">
				<Image Source="university.png" HeightRequest="80" WidthRequest="80"
					   SemanticProperties.Description="Northbridge University emblem" />
				<Label Text="Northbridge University" Style="{StaticResource PageHeading}" />
				<Label Text="Welcome to your student campus companion."
					   Style="{StaticResource BodyText}" />
				<Button Text="Browse courses" AutomationId="BrowseCoursesButton"
						Style="{StaticResource ActionButton}" Command="{Binding BrowseCoursesCommand}" />
			</VerticalStackLayout>
		</ScrollView>
	</Grid>
</ContentPage>
```

### 5.4 Views/CoursesPage.xaml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage x:Class="UniversityNavigator.App.Views.CoursesPage"
			 x:DataType="CoursesViewModel"
			 Style="{StaticResource UniversityPage}" Title="Courses">
	<Grid Padding="16" RowDefinitions="Auto,*,Auto" RowSpacing="12">
		<Label Text="Course catalog" Style="{StaticResource PageHeading}" />
		<CollectionView Grid.Row="1" ItemsSource="{Binding Courses}"
						SelectedItem="{Binding SelectedCourse, Mode=TwoWay}"
						SelectionMode="Single" AutomationId="CourseList">
			<CollectionView.ItemTemplate>
				<DataTemplate x:DataType="Course">
					<Border Margin="0,4" Padding="12" Stroke="#757575"
							StrokeShape="RoundRectangle 6"
							Background="{AppThemeBinding Light=#F4F7F5, Dark=#262626}">
						<VerticalStackLayout Spacing="4">
							<Label Text="{Binding Id}" FontAttributes="Bold"
								   Style="{StaticResource BodyText}" />
							<Label Text="{Binding Title}" Style="{StaticResource BodyText}" />
							<Label Text="{Binding Credits, StringFormat='Credits: {0}'}"
								   Style="{StaticResource BodyText}" />
						</VerticalStackLayout>
					</Border>
				</DataTemplate>
			</CollectionView.ItemTemplate>
		</CollectionView>
		<VerticalStackLayout Grid.Row="2" Spacing="8">
			<Label Text="{Binding ResultMessage}" AutomationId="FavoriteResult"
				   Style="{StaticResource BodyText}" />
			<Button Text="View course by ID" Command="{Binding OpenByIdCommand}"
					AutomationId="OpenCourseByIdButton" Style="{StaticResource ActionButton}" />
			<Button Text="View course by object" Command="{Binding OpenByObjectCommand}"
					AutomationId="OpenCourseByObjectButton" Style="{StaticResource ActionButton}" />
			<Button Text="View course by path" Command="{Binding OpenByPathCommand}"
					AutomationId="OpenCourseByPathButton" Style="{StaticResource ActionButton}" />
		</VerticalStackLayout>
	</Grid>
</ContentPage>
```

Select a course, then choose an opening method. The three buttons are learning variants, not a recommended final product design. A production catalog would normally expose one primary detail action. The global mapping identifies the Core assembly once, so the item template can use `Course` without a page-level model namespace declaration. Its `x:DataType` describes one course item, not the page's CoursesViewModel.

### 5.5 Views/CourseDetailsPage.xaml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage x:Class="UniversityNavigator.App.Views.CourseDetailsPage"
			 x:DataType="CourseDetailsViewModel"
			 Style="{StaticResource UniversityPage}" Title="Course details">
	<Shell.BackButtonBehavior>
		<BackButtonBehavior Command="{Binding BackCommand}"
							AccessibilityLabel="Back to course catalog" />
	</Shell.BackButtonBehavior>
	<Grid>
		<ScrollView>
			<VerticalStackLayout Padding="24" Spacing="16">
				<Label Text="{Binding Course.Title}" AutomationId="CourseTitle"
					   Style="{StaticResource PageHeading}" />
				<Label Text="{Binding Course.Id}" Style="{StaticResource BodyText}" />
				<Label Text="{Binding Course.Description}" Style="{StaticResource BodyText}" />
				<Label Text="{Binding Status}" AutomationId="CourseStatus"
					   Style="{StaticResource BodyText}" />
				<Button Text="Meet the instructor" Command="{Binding OpenInstructorCommand}"
						AutomationId="OpenInstructorButton" Style="{StaticResource ActionButton}" />
				<Button Text="Choose as favorite" Command="{Binding ChooseFavoriteCommand}"
						AutomationId="ChooseFavoriteButton" Style="{StaticResource ActionButton}" />
				<Button Text="Return to catalog root" Command="{Binding CatalogRootCommand}"
						AutomationId="CatalogRootButton" Style="{StaticResource ActionButton}" />
			</VerticalStackLayout>
		</ScrollView>
	</Grid>
</ContentPage>
```

`BackButtonBehavior` customizes the toolbar Back button, not every platform gesture or hardware Back action. Cross-cutting rules belong in Shell's navigation guard, which you will add below.

### 5.6 Views/InstructorDetailsPage.xaml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage x:Class="UniversityNavigator.App.Views.InstructorDetailsPage"
			 x:DataType="InstructorDetailsViewModel"
			 Style="{StaticResource UniversityPage}" Title="Instructor">
	<Grid>
		<ScrollView>
			<VerticalStackLayout Padding="24" Spacing="16">
				<Label Text="{Binding Instructor.Name}" AutomationId="InstructorName"
					   Style="{StaticResource PageHeading}" />
				<Label Text="{Binding Instructor.Office}" Style="{StaticResource BodyText}" />
				<Label Text="{Binding Instructor.Email}" Style="{StaticResource BodyText}" />
				<Button Text="Back to course" Command="{Binding BackCommand}"
						Style="{StaticResource ActionButton}" />
				<Button Text="Back two pages to catalog" Command="{Binding BackTwoLevelsCommand}"
						AutomationId="BackTwoLevelsButton" Style="{StaticResource ActionButton}" />
			</VerticalStackLayout>
		</ScrollView>
	</Grid>
</ContentPage>
```

### 5.7 Views/CampusPage.xaml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage x:Class="UniversityNavigator.App.Views.CampusPage"
			 x:DataType="CampusViewModel"
			 Style="{StaticResource UniversityPage}" Title="Campus">
	<Grid>
		<ScrollView>
			<VerticalStackLayout Padding="24" Spacing="16">
				<Label Text="Student services" Style="{StaticResource PageHeading}" />
				<Label Text="Visit the Welcome Centre, Monday to Friday, 9:00-17:00."
					   Style="{StaticResource BodyText}" />
				<Button Text="Campus help" Command="{Binding OpenHelpCommand}"
						AutomationId="OpenHelpButton" Style="{StaticResource ActionButton}" />
				<Button Text="University website" Command="{Binding OpenWebsiteCommand}"
						Style="{StaticResource ActionButton}" />
				<Button Text="Email student services" Command="{Binding EmailServicesCommand}"
						Style="{StaticResource ActionButton}" />
				<Label Text="{Binding ExternalStatus}" Style="{StaticResource BodyText}" />
			</VerticalStackLayout>
		</ScrollView>
	</Grid>
</ContentPage>
```

### 5.8 Views/CampusHelpPage.xaml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage x:Class="UniversityNavigator.App.Views.CampusHelpPage"
			 x:DataType="CampusHelpViewModel"
			 Shell.PresentationMode="ModalAnimated"
			 Style="{StaticResource UniversityPage}" Title="Campus help">
	<Grid>
		<ScrollView>
			<VerticalStackLayout Padding="24" Spacing="16">
				<Label Text="Welcome Centre" AutomationId="CampusHelpHeading"
					   Style="{StaticResource PageHeading}" />
				<Label Text="Bring your student ID for timetable, access, or registration assistance."
					   Style="{StaticResource BodyText}" />
				<Button Text="Close" Command="{Binding CloseCommand}"
						AutomationId="CloseHelpButton" Style="{StaticResource ActionButton}" />
			</VerticalStackLayout>
		</ScrollView>
	</Grid>
</ContentPage>
```

A modal interrupts the current workflow and has an explicit completion or dismissal action. Do not depend on a toolbar Back button being supplied for a modal. `ModalAnimated` requests a modal transition; sheet versus full-screen appearance is platform-dependent. Presentation alternatives include `Animated`, `NotAnimated`, and `ModalNotAnimated`.

### 5.9 Views/ProfilePage.xaml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage x:Class="UniversityNavigator.App.Views.ProfilePage"
			 x:DataType="ProfileViewModel"
			 Style="{StaticResource UniversityPage}" Title="Profile">
	<Grid>
		<ScrollView>
			<VerticalStackLayout Padding="24" Spacing="16">
				<Label Text="{Binding Student.Name}" AutomationId="StudentName"
					   Style="{StaticResource PageHeading}" />
				<Label Text="{Binding Student.StudentId}" Style="{StaticResource BodyText}" />
				<Label Text="{Binding Student.Program}" Style="{StaticResource BodyText}" />
				<Button Text="Edit profile" Command="{Binding EditCommand}"
						AutomationId="EditProfileButton" Style="{StaticResource ActionButton}" />
			</VerticalStackLayout>
		</ScrollView>
	</Grid>
</ContentPage>
```

### 5.10 Views/EditProfilePage.xaml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage x:Class="UniversityNavigator.App.Views.EditProfilePage"
			 x:DataType="EditProfileViewModel"
			 Style="{StaticResource UniversityPage}" Title="Edit profile">
	<Grid>
		<ScrollView>
			<VerticalStackLayout Padding="24" Spacing="16">
				<Label Text="Student name" Style="{StaticResource PageHeading}" />
				<Entry Text="{Binding Name, Mode=TwoWay}" Placeholder="Full name"
					   AutomationId="StudentNameEntry" SemanticProperties.Description="Student full name"
					   Background="{AppThemeBinding Light=#FFFFFF, Dark=#262626}"
					   TextColor="{AppThemeBinding Light=#202020, Dark=#F2F2F2}"
					   PlaceholderColor="{AppThemeBinding Light=#595959, Dark=#C7C7C7}" />
				<Label Text="{Binding ValidationMessage}" AutomationId="ProfileValidation"
					   Style="{StaticResource BodyText}" />
				<Button Text="Save" Command="{Binding SaveCommand}"
						AutomationId="SaveProfileButton" Style="{StaticResource ActionButton}" />
				<Button Text="Cancel" Command="{Binding CancelCommand}"
						AutomationId="CancelProfileButton" Style="{StaticResource ActionButton}" />
			</VerticalStackLayout>
		</ScrollView>
	</Grid>
</ContentPage>
```

The ScrollView allows the editor actions to remain reachable when the keyboard reduces the available height. Explicit color tokens express intent only; verify actual native Entry rendering in both themes with DevFlow.

## 6. Add the tab icons and university emblem

Create the following five files inside the app's existing `Resources/Images` folder. These simple SVG sources need no download or icon package. Keep the template's `MauiImage Include="Resources\Images\*"` entry; it includes these files. If your template lacks that wildcard, add it once inside an `ItemGroup`.

`home.svg`:

```xml
<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24">
  <path d="M3 11L12 3L21 11V21H15V14H9V21H3Z" fill="#185B46" />
</svg>
```

`courses.svg`:

```xml
<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24">
  <path d="M3 4H10L12 6L14 4H21V20H14L12 22L10 20H3Z" fill="none" stroke="#185B46" stroke-width="2" />
  <path d="M12 6V21" stroke="#185B46" stroke-width="2" />
</svg>
```

`campus.svg`:

```xml
<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24">
  <path d="M2 8L12 2L22 8ZM2 20H22V23H2ZM5 10H8V18H5ZM11 10H14V18H11ZM17 10H20V18H17Z" fill="#185B46" />
</svg>
```

`profile.svg`:

```xml
<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24">
  <circle cx="12" cy="7" r="4" fill="#185B46" />
  <path d="M4 22V18C4 11 20 11 20 18V22Z" fill="#185B46" />
</svg>
```

`university.svg`:

```xml
<svg xmlns="http://www.w3.org/2000/svg" width="96" height="96" viewBox="0 0 96 96">
  <path d="M12 8H84V54Q84 76 48 90Q12 76 12 54Z" fill="#185B46" />
  <path d="M26 25H38L58 57V25H70V72H58L38 40V72H26Z" fill="#FFFFFF" />
</svg>
```

Reference these assets as `home.png`, `courses.png`, `campus.png`, `profile.png`, and `university.png` in XAML. MAUI converts SVG sources to PNG resources during the build. Do not create another PNG with the same base name or reference `.svg` at runtime. Native tab icons may be tinted by Shell.

## 7. Wire Shell, dependency injection, and DevFlow

### 7.1 Replace AppShell.xaml

> Normally I would configure my Shell / Tab navigation using XAML like the code below, but there appears to be a bug with the combination of pre-release packages that we are using.  The code below is just a reference of how you would typically configure in XAML.  For this assignment, due to the bug, we will add an empty Shell element to the AppShell.xaml and then build out the Shell in the AppShell.xaml.cs C# code behind page.  DO NOT USE THE XAML below this note.  

**The below code is just for reference.  Do not use it in this assignment.**

One TabBar with four Tabs produces four main destinations. Each Tab contains exactly one ShellContent. Multiple ShellContent children inside a Tab would introduce another level of navigation, commonly top tabs; do not add them to the submitted main layout.

Explicit routes keep addresses predictable. A XAML filename is not automatically a registered route. ShellContent uses a DataTemplate so tab content can be created on demand, rather than eagerly constructing every page at startup.

```xml
<!-- DO NOT USE THIS APPSHELL XAML CODE -->
<?xml version="1.0" encoding="utf-8" ?>
<Shell x:Class="UniversityNavigator.App.AppShell"
	   FlyoutBehavior="Disabled"
	   Shell.BackgroundColor="{AppThemeBinding Light=#FFFFFF, Dark=#161616}"
	   Shell.ForegroundColor="{AppThemeBinding Light=#185B46, Dark=#B8E4D3}"
	   Shell.TitleColor="{AppThemeBinding Light=#242424, Dark=#F2F2F2}"
	   Shell.TabBarBackgroundColor="{AppThemeBinding Light=#FFFFFF, Dark=#161616}"
	   Shell.TabBarForegroundColor="{AppThemeBinding Light=#185B46, Dark=#B8E4D3}"
	   Shell.TabBarUnselectedColor="{AppThemeBinding Light=#595959, Dark=#C7C7C7}">
	<TabBar Route="university">
		<Tab Title="Home" Route="home" Icon="home.png">
			<ShellContent Route="welcome" ContentTemplate="{DataTemplate HomePage}" />
		</Tab>
		<Tab Title="Courses" Route="courses" Icon="courses.png">
			<ShellContent Route="catalog" ContentTemplate="{DataTemplate CoursesPage}" />
		</Tab>
		<Tab Title="Campus" Route="campus" Icon="campus.png">
			<ShellContent Route="services" ContentTemplate="{DataTemplate CampusPage}" />
		</Tab>
		<Tab Title="Profile" Route="profile" Icon="profile.png">
			<ShellContent Route="student" ContentTemplate="{DataTemplate ProfilePage}" />
		</Tab>
	</TabBar>
</Shell>
```

**USE THIS APPSHELL XAML CODE**

For this assignment, replace the content of AppShell.xaml with the following:
```xml

<?xml version="1.0" encoding="utf-8" ?>
<Shell x:Class="UniversityNavigator.App.AppShell"
	   FlyoutBehavior="Disabled"
	   Shell.BackgroundColor="{AppThemeBinding Light=#FFFFFF, Dark=#161616}"
	   Shell.ForegroundColor="{AppThemeBinding Light=#185B46, Dark=#B8E4D3}"
	   Shell.TitleColor="{AppThemeBinding Light=#242424, Dark=#F2F2F2}"
	   Shell.TabBarBackgroundColor="{AppThemeBinding Light=#FFFFFF, Dark=#161616}"
	   Shell.TabBarForegroundColor="{AppThemeBinding Light=#185B46, Dark=#B8E4D3}"
	   Shell.TabBarUnselectedColor="{AppThemeBinding Light=#595959, Dark=#C7C7C7}" />

```



### 7.2 Replace AppShell.xaml.cs

```csharp
using System.Diagnostics;
using UniversityNavigator.App.ViewModels;
using UniversityNavigator.App.Views;

namespace UniversityNavigator.App;

public partial class AppShell : Shell
{
	private bool confirmingNavigation;

	public AppShell(IServiceProvider services)
	{
		InitializeComponent();
		Items.Add(CreateTabBar(services));
		Routing.RegisterRoute("course-details", typeof(CourseDetailsPage));
		Routing.RegisterRoute("instructor-details", typeof(InstructorDetailsPage));
		Routing.RegisterRoute("campus-help", typeof(CampusHelpPage));
		Routing.RegisterRoute("edit-profile", typeof(EditProfilePage));
		Routing.RegisterRoute("course/{pathCourseId}", typeof(CourseDetailsPage));
	}

	private static TabBar CreateTabBar(IServiceProvider services)
	{
		var tabBar = new TabBar { Route = "university" };
		tabBar.Items.Add(CreateTab<HomePage>(services, "Home", "home", "home.png", "welcome"));
		tabBar.Items.Add(CreateTab<CoursesPage>(services, "Courses", "courses", "courses.png", "catalog"));
		tabBar.Items.Add(CreateTab<CampusPage>(services, "Campus", "campus", "campus.png", "services"));
		tabBar.Items.Add(CreateTab<ProfilePage>(services, "Profile", "profile", "profile.png", "student"));
		return tabBar;
	}

	private static Tab CreateTab<TPage>(
		IServiceProvider services,
		string title,
		string route,
		string icon,
		string contentRoute)
		where TPage : Page
	{
		var tab = new Tab { Title = title, Route = route, Icon = icon };
		tab.Items.Add(new ShellContent
		{
			Route = contentRoute,
			ContentTemplate = new DataTemplate(() => services.GetRequiredService<TPage>())
		});
		return tab;
	}

	protected override async void OnNavigating(ShellNavigatingEventArgs args)
	{
		base.OnNavigating(args);

		if (!args.CanCancel)
			return;

		if (confirmingNavigation)
		{
			args.Cancel();
			return;
		}

		if (CurrentPage?.BindingContext is not EditProfileViewModel editor || !editor.IsDirty)
			return;

		var deferral = args.GetDeferral();
		confirmingNavigation = true;
		try
		{
			bool discard = await CurrentPage.DisplayAlertAsync(
				"Discard changes?", "Your student name has not been saved.", "Discard", "Stay");

			if (discard)
				editor.DiscardChanges();
			else
				args.Cancel();
		}
		catch (Exception exception)
		{
			args.Cancel();
			Debug.WriteLine($"Navigation confirmation failed: {exception.GetType().Name}");
		}
		finally
		{
			confirmingNavigation = false;
			deferral.Complete();
		}
	}

	protected override void OnNavigated(ShellNavigatedEventArgs args)
	{
		base.OnNavigated(args);
#if DEBUG
		Debug.WriteLine($"Navigation: {args.Source}; {args.Current.Location}");
		Debug.WriteLine($"Pages: {string.Join(" -> ", Navigation.NavigationStack.Select(page => page?.GetType().Name ?? "(Shell root)"))}");
		Debug.WriteLine($"Modal count: {Navigation.ModalStack.Count}");
#endif
	}
}

```

Register routes once, before navigation. Do not also register `catalog`, `welcome`, `services`, or `student` with `Routing.RegisterRoute`: they already belong to the visual hierarchy. Do not give two different destinations the same route.

The guard waits for the alert through a **deferral**, then completes or cancels the original navigation. `finally` releases the deferral even if the dialog fails. Do not call `GoToAsync` inside this pending deferral; Shell rejects overlapping navigation while it waits. `async void` is appropriate here only because this is a framework event-style override; command methods return Task.

The guard covers cancellable Shell navigation away from the editor, including tab changes and Back requests routed through Shell. It is not persistence and cannot prevent OS process termination. Native gestures can differ by platform; testing them is part of the assignment.

Navigation logging is debug-only. In real applications, URLs and query strings can contain personal data: do not log tokens, passwords, or sensitive student information.

### 7.3 Replace MauiProgram.cs

```csharp
using Microsoft.Extensions.Logging;
using UniversityNavigator.App.ViewModels;
using UniversityNavigator.App.Views;
using UniversityNavigator.Core.Services;
#if MAUI_DEVFLOW
using Microsoft.Maui.DevFlow.Agent;
#endif

namespace UniversityNavigator.App;

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

#if DEBUG
		builder.Logging.AddDebug();
#endif
#if MAUI_DEVFLOW
		builder.AddMauiDevFlowAgent();
#endif

		builder.Services.AddSingleton<CourseCatalog>();
		builder.Services.AddSingleton<AppShell>();

		builder.Services.AddSingleton<HomeViewModel>();
		builder.Services.AddSingleton<CoursesViewModel>();
		builder.Services.AddSingleton<CampusViewModel>();
		builder.Services.AddSingleton<ProfileViewModel>();
		builder.Services.AddTransient<HomePage>();
		builder.Services.AddTransient<CoursesPage>();
		builder.Services.AddTransient<CampusPage>();
		builder.Services.AddTransient<ProfilePage>();

		builder.Services.AddTransient<CourseDetailsViewModel>();
		builder.Services.AddTransient<InstructorDetailsViewModel>();
		builder.Services.AddTransient<CampusHelpViewModel>();
		builder.Services.AddTransient<EditProfileViewModel>();
		builder.Services.AddTransient<CourseDetailsPage>();
		builder.Services.AddTransient<InstructorDetailsPage>();
		builder.Services.AddTransient<CampusHelpPage>();
		builder.Services.AddTransient<EditProfilePage>();

		return builder.Build();
	}
}
```

Keep the two font files generated by the template. Shell uses registered services when creating these pages. Registering both pages and their constructor dependencies prevents "no suitable constructor" activation errors.

The catalog and root view models are singletons for this single-window teaching app: selected courses and a saved profile remain available while the process runs. Detail view models are transient so two detail visits do not accidentally share the same mutable selection or draft. A transient page already on the navigation stack is not recreated merely because another page above it is popped. Singleton is not permanent storage and is not automatically appropriate for a multi-window app.

### 7.4 Replace App.xaml.cs

```csharp
namespace UniversityNavigator.App;

public partial class App : Application
{
	private readonly AppShell shell;

	public App(AppShell shell)
	{
		InitializeComponent();
		this.shell = shell;
	}

	protected override Window CreateWindow(IActivationState? activationState) => new(shell);
}
```

The window starts with Shell exactly once. Do not navigate by repeatedly replacing the window's Page or the obsolete Application.MainPage property. Delete the unused generated `MainPage.xaml` and `MainPage.xaml.cs` now, after verifying that AppShell no longer references them.

### 7.5 Enable DevFlow for debugger launches

1. Use the current Microsoft .NET MAUI VS Code extension with its DevFlow-enabled debugging workflow. Its debug build injects the compatible DevFlow package and the `MAUI_DEVFLOW` symbol when `MauiDevFlowEnabled=true`.
2. Keep the conditional using and registration above. An ordinary build without DevFlow can compile without the agent package. Do not define `MAUI_DEVFLOW` manually without the build integration that supplies the package.
3. Do not add a permanently pinned `Microsoft.Maui.DevFlow.Agent` package to this extension-managed setup. There is no extra agent package command to run here; the extension supplies it. If your installed extension lacks this integration, update the extension before continuing.
4. If testing Mac Catalyst, open `Platforms/MacCatalyst/Entitlements.plist`. Inside its existing root `dict`, add the following key only if it is missing. Preserve any existing entitlements:

```xml
<key>com.apple.security.network.server</key>
<true/>
```

5. Launch under the MAUI debugger, wait for the DevFlow agent connection, and inspect the visual tree. If no agent connects, check the selected startup project, debug output, and extension DevFlow diagnostics before changing package references. An empty agent list is not proof that registration code is absent.

This is native MAUI, not Blazor Hybrid, so it needs neither BlazorWebView nor Blazor DevFlow packages.

### Checkpoint B: four tabs and a detail journey

Launch the selected MAUI app on your mobile target. If a debug session is already active, try Hot Reload first; restart through the MAUI debugger if the new types or startup changes cannot be applied.

1. Confirm Home, Courses, Campus, and Profile appear with distinct icons.
2. From Home, tap Browse courses. Confirm the selected tab is Courses.
3. Select COMP301 and tap View course by ID. Confirm the title is Mobile Application Development.
4. Tap Meet the instructor. Confirm Dr. Maya Chen appears.
5. Use Back to course, then the native toolbar Back button. Confirm the original catalog returns.
6. Repeat with HIST210 and View course by object. Confirm the title containing `&` renders correctly.
7. Open Campus help and close it. Confirm you return to Campus, not Home.

Do not proceed until this basic journey works. The next sections explain and test the different navigation contracts rather than adding more application infrastructure.

## 8. Compare the data-passing techniques

### 8.1 Pass a small identifier

1. Select HIST210 and choose View course by ID.
2. Put a breakpoint in `CourseDetailsViewModel.ApplyQueryAttributes`.
3. Inspect the `courseId` key and the resulting Course.
4. Return to the catalog, select BIOL120, and repeat.

The route carries a small string; the detail view model loads the matching object. This is the preferred pattern for links that must be reconstructible after app restart. An ID is not authorization: a real backend must still check whether a student may access the requested record.

Temporarily change the ID command's route to `"course-details?courseId=UNKNOWN"`. Verify the error message and disabled instructor/favorite actions, then restore the original command. Also test `"course-details"` with no parameter. These are deliberate invalid-input tests, not additional registered routes.

### 8.2 Pass an existing object

1. Choose View course by object.
2. At the same breakpoint, inspect the `course` key. It contains a Course instance, not JSON.
3. Open the instructor and return. Confirm the course is still displayed even though its single-use input is not delivered again.

`ShellNavigationQueryParameters` clears its own data after navigation. The destination can still retain the object in its Course property. Clearing the parameter collection does not dispose or erase the destination's object reference.

For comparison, replace just the parameter construction in `OpenByObjectAsync` temporarily with:

```csharp
new Dictionary<string, object> { ["course"] = course }
```

Repeat the instructor round trip and inspect calls to `ApplyQueryAttributes`. A regular dictionary is retained with the destination's navigation data for the page's lifetime and can be delivered again when navigating back. This can unexpectedly reapply old input or retain a large object graph. Restore `ShellNavigationQueryParameters` afterward. Do not retain entire service providers, pages, images, or sensitive records as navigation arguments.

### 8.3 Return a result to the parent

1. Open a course and choose Choose as favorite.
2. Confirm the existing CoursesPage displays the returned ID's course title.
3. Inspect the route and stack: the detail page was popped; a second CoursesPage was not pushed.
4. Edit the profile and Save. Confirm the existing ProfilePage updates immediately.
5. Reopen the editor, change the name, Cancel, then choose Discard. Confirm the parent's saved name is unchanged.

The favorite example returns a query string with `..?favoriteCourseId=...`. The profile example returns a typed object with `GoToAsync("..", parameters)`. A shared singleton service, observable shared model, or a message bus can also coordinate state, but those are state-sharing mechanisms, not new types of page navigation. Do not introduce a global static `SelectedCourse` just to avoid passing an argument.

### 8.4 Recognize QueryProperty, but prefer IQueryAttributable

You will encounter this alternative in tutorials. The following is a **comparison only**, not a replacement to paste into the working application:

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

namespace UniversityNavigator.App.ViewModels;

[QueryProperty(nameof(CourseId), "courseId")]
public partial class AttributeExampleViewModel : ObservableObject
{
	[ObservableProperty]
	private string courseId = "";
}
```

`QueryProperty` maps an incoming key to a property and automatically URL-decodes query-string values. It is not safe for full trimming or NativeAOT. The working app therefore uses `IQueryAttributable`, which also lets you validate several parameters together. Do not implement both approaches for the same key on both the page and its view model.

## 9. Learn route forms and navigation-stack management

### 9.1 Route reference for this application

| Request | Meaning and precondition |
| --- | --- |
| `//university/home/welcome` | Select the Home root |
| `//university/courses/catalog` | Select Courses and request its root, removing pushed detail history for that destination |
| `course-details?courseId=COMP301` | Push a registered course detail on the current stack; our UI invokes it from Courses |
| `instructor-details` with an object | Push an instructor page above the course |
| `..` | Pop one page; requires a page to return to |
| `../..` | Pop two pages; requires two pushed levels |
| `..?favoriteCourseId=COMP301` | Pop and deliver a result to the previous page |
| `../course-details?courseId=BIOL120` | Pop once, then push a course detail from the resulting context |
| `//university/courses/catalog/course/COMP301` | .NET 11 absolute template route, anchored at a real Shell root |
| `//course-details` | Invalid: a registered global detail route cannot be the sole root |

Do not treat `GoToAsync("courses")` as a general way to switch to a declared tab. Use the explicit absolute hierarchy address. Conversely, do not prepend `//` to an ordinary detail route without a visual-hierarchy anchor.

`await Shell.Current.GoToAsync("course-details?courseId=COMP301", animate: false);` performs the same push without the animation. Changing animation does not change history.

### 9.2 Observe tab history versus a deliberate reset

1. From Courses, open a course and then its instructor.
2. Tap Campus, then tap Courses in the tab bar. Observe whether your platform restores the instructor and record the route and stack. Ordinary tab selection is intended to preserve that tab's history; platform tab reselection behavior can differ.
3. Use Back two pages to catalog. Confirm both pushed detail levels are removed.
4. Repeat the course/instructor journey. This time go Home and press Browse courses. It requests the Courses root explicitly. Record the difference from tapping the Courses tab.
5. From a course detail, use Return to catalog root. Confirm Back no longer returns to that detail.

Resetting one destination is not a global reset of every tab, every singleton view model, or the modal stack. Do not describe this as a complete logout implementation. A real logout must clear sensitive state and credentials and apply an intentional navigation policy for all reachable destinations.

### 9.3 Inspect the actual stack

Pause the debugger after navigation completes. Inspect these expressions:

```csharp
Shell.Current.CurrentState.Location.ToString()
Shell.Current.CurrentPage.GetType().Name
Shell.Current.Navigation.NavigationStack
Shell.Current.Navigation.ModalStack
Shell.Current.CurrentItem.CurrentItem.Stack
```

The last expression inspects the selected Shell section's stack. These collections are read-only views, not lists you should clear or modify. Shell's root representation can include an implicit entry; report the actual entries rather than assuming a count of zero or one for every root. Use the application's debug output to associate a route with `Push`, `Pop`, or tab-change events.

Draw the conceptual stack before and after each operation:

| Before | Operation | After |
| --- | --- | --- |
| `[Catalog]` | Open course | `[Catalog, Course]` |
| `[Catalog, Course]` | Open instructor | `[Catalog, Course, Instructor]` |
| `[Catalog, Course, Instructor]` | `..` | `[Catalog, Course]` |
| `[Catalog, Course, Instructor]` | `../..` | `[Catalog]` |
| `[Catalog, Course]` | Absolute catalog root | `[Catalog]` |
| Campus with no modal | Open help | Campus remains underneath a modal |
| Campus with help modal | Close | Campus with no modal |

Never implement Back with `GoToAsync("course-details")`: that pushes another detail page. Never call `..` repeatedly at a root hoping it selects a different tab. On Android, system Back at a root can leave the app instead of visiting previously selected tabs.

### 9.4 Replace the current detail using routes

As a short experiment, add this command to CourseDetailsViewModel and a button bound to `NextCourseCommand` inside that page's existing VerticalStackLayout:

```csharp
[RelayCommand]
private async Task NextCourseAsync() =>
	await Shell.Current.GoToAsync("../course-details?courseId=BIOL120");
```

Open COMP301 from the catalog, invoke the new button, and press Back. You should return to the catalog, not COMP301: the old detail was popped before the new detail was pushed. Remove this experimental command and button after recording the result. By contrast, navigating directly to `course-details?courseId=BIOL120` from a detail would add another detail level.

### 9.5 Contextual navigation

For a separate experiment, add this registration after the others in AppShell's constructor:

```csharp
Routing.RegisterRoute("catalog/context-details", typeof(CourseDetailsPage));
```

Temporarily change the relative route in `OpenByIdAsync` from `course-details` to `context-details`, leaving its query string unchanged. Run it from the catalog root. Shell resolves the route in the current catalog context. This allows features to use local detail route names without assuming every detail is a global destination.

Do not additionally register a global `context-details` alias during this experiment, and do not expect the contextual alias to resolve from arbitrary pages. Restore `OpenByIdAsync` and remove the experimental registration after testing. A route table should not accumulate unused aliases.

### 9.6 .NET 11 route templates

Choose View course by path. The registration `course/{pathCourseId}` captures COMP301 from the path and delivers it to the same detail view model. The destination receiver was included in Step 4.

Important differences from query-string routes:

- Template navigation requires an **absolute** URI beginning at a Shell visual-hierarchy route. `course/COMP301` alone is not a supported relative template request.
- `//course/COMP301` is not enough: the registered detail route is not a standalone Shell root.
- Captured path values are strings, even when a constraint such as `{number:int}` is used.
- Path values are already URL-decoded when received through `IQueryAttributable`.
- Templates can support required, optional, defaulted, constrained, catch-all, and mixed segments. Learn the required-ID form first; validation still belongs in the receiver.

Explain why the query-string and path buttons use different receiver keys even though both locate a course by ID. Test UNKNOWN with the absolute template route and confirm the same recoverable error state. If the template is rejected by an earlier preview, use the SDK/workload requirement in Step 2 rather than changing it into an undocumented relative route.

## 10. Test modal behavior and unsaved changes

### 10.1 Modal exercise

1. Select Campus, open help, and inspect `ModalStack`.
2. Confirm the help page is visible and the underlying Campus page has not become another copy.
3. Close the modal and inspect the stack again.
4. Reopen it and test the platform Back/dismiss gesture if available. Record any difference from the explicit Close button.

An alert such as `DisplayAlertAsync` is a dialog, not a routable ContentPage. An action sheet is a menu of choices, not a tab. Use these UI affordances for decisions; use a modal page for a focused workflow that needs page content.

### 10.2 Guard exercise

Test each case from Edit profile, starting with a fresh draft where appropriate:

| Action | Expected result |
| --- | --- |
| Cancel without editing | Return immediately, no warning |
| Change name, Cancel, Stay | Remain in editor with draft intact |
| Change name, Cancel, Discard | Return; parent's saved name unchanged |
| Change name, toolbar Back, Stay | Same protection as Cancel |
| Change name, Android Back or available iOS Back gesture | Verify actual platform behavior and guard coverage |
| Change name, tap another tab, Stay | Navigation canceled; editor remains selected |
| Change name, tap another tab, Discard | Tab changes; if editor is retained in Profile history, its draft is reset |
| Clear name and Save | Validation message; remain in editor |
| Enter valid name and Save | Return with updated profile, no discard warning |

Hiding a Back button is not a navigation guard. Disabling toolbar Back does not prevent every OS gesture, tab selection, or programmatic route. Never rely on a confirmation dialog as the only protection against losing important data when the operating system closes the process.

## 11. Compare Shell flyout and top-tab layouts

These are temporary experiments. The submitted main app must finish with the original four bottom tabs. Record your results, then restore AppShell exactly as it was in Step 7.

### 11.1 Shell flyout alternative

1. Stop debugging before changing the navigation container declaration.
2. Change Shell's `FlyoutBehavior` to `Flyout`.
3. Replace the opening `<TabBar Route="university">` with the following tag, and replace its closing tag with `</FlyoutItem>`. Keep all four existing Tab children unchanged.

```xml
<FlyoutItem Route="university" Title="Northbridge" FlyoutDisplayOptions="AsMultipleItems">
```

4. Launch and open the flyout. Record how its destinations differ from the original TabBar. Because route names were retained, test that the Home shortcut still works.
5. Restore `TabBar` and `FlyoutBehavior="Disabled"`.

Shell's FlyoutItem is part of Shell. It is **not** a FlyoutPage nested inside Shell. Flyout navigation can suit many destinations; four frequently used student destinations are easier to reach with bottom tabs.

### 11.2 Secondary/top tabs

1. Inside the Campus Tab, temporarily add a second ShellContent after `services`:

```xml
<ShellContent Title="Welcome" Route="campus-welcome"
			  ContentTemplate="{DataTemplate HomePage}" />
```

2. Set `Title="Services"` on the existing Campus ShellContent.
3. Launch and observe the secondary tabs for the selected Campus tab. Placement follows platform conventions.
4. Explain why these are not a fifth bottom tab, then remove the extra ShellContent and restore the original.

## 12. Non-Shell comparison sandbox

Complete this lab in a **separate project**, never inside the working Shell app. It demonstrates navigation techniques you will encounter in non-Shell MAUI apps. The sandbox is intentionally small and uses code-created controls so the comparison stays focused on navigation APIs.

### 12.1 Create and prepare the sandbox

From the solution root:

```console
dotnet new maui --name UniversityNavigator.NavigationLab --output src/UniversityNavigator.NavigationLab --framework net11.0
dotnet sln UniversityNavigator.slnx add src/UniversityNavigator.NavigationLab/UniversityNavigator.NavigationLab.csproj
```

No project reference or additional NuGet package is required: this lab has no shared model dependency. Copy the four tab SVG files from the main app's Resources/Images folder into the lab's Resources/Images folder using VS Code Explorer. Add the same conditional DevFlow using and `AddMauiDevFlowAgent()` call from Step 7 to the lab's generated MauiProgram. For Mac Catalyst, add the same entitlement if needed.

Keep the generated App.xaml resource dictionaries and MauiProgram's font registration. You will replace only App.xaml.cs and add the file below. The generated AppShell and MainPage can remain unused; do not instantiate AppShell in the lab's window.

### 12.2 Add NavigationLabPages.cs at the lab project root

```csharp
namespace UniversityNavigator.NavigationLab;

public static class NavigationLabPages
{
	private static ContentPage CreatePage(string title, out VerticalStackLayout layout)
	{
		layout = new VerticalStackLayout { Padding = 24, Spacing = 16 };
		layout.Add(new Label { Text = title, FontSize = 24, TextColor = Colors.Black });
		var grid = new Grid();
		grid.Add(new ScrollView { Content = layout });
		return new ContentPage { Title = title, Background = Colors.White, Content = grid };
	}

	private static void AddAction(ContentPage page, VerticalStackLayout layout,
		string text, Func<Task> action)
	{
		var button = new Button
		{
			Text = text,
			Background = Color.FromArgb("#185B46"),
			TextColor = Colors.White,
			MinimumHeightRequest = 48
		};
		button.Clicked += async (_, _) =>
		{
			button.IsEnabled = false;
			try
			{
				await action();
			}
			catch (Exception exception)
			{
				await page.DisplayAlertAsync("Navigation lab", exception.Message, "OK");
			}
			finally
			{
				button.IsEnabled = true;
			}
		};
		layout.Add(button);
	}

	public static ContentPage CreateCatalog()
	{
		var page = CreatePage("University course catalog", out var layout);
		AddAction(page, layout, "Open COMP301", async () =>
			await page.Navigation.PushAsync(CreateDetails("COMP301")));
		AddAction(page, layout, "Open campus help modal", async () =>
			await page.Navigation.PushModalAsync(CreateHelp()));
		return page;
	}

	private static ContentPage CreateDetails(string courseId)
	{
		var page = CreatePage($"Course: {courseId}", out var layout);
		AddAction(page, layout, "Back", async () => await page.Navigation.PopAsync());
		AddAction(page, layout, "Open registration review", async () =>
			await page.Navigation.PushAsync(CreateReview(page)));
		AddAction(page, layout, "Return to root", async () =>
			await page.Navigation.PopToRootAsync());
		AddAction(page, layout, "Insert advising before this page", async () =>
		{
			var advising = CreatePage("Academic advising", out var advisingLayout);
			AddAction(advising, advisingLayout, "Back to catalog", async () =>
				await advising.Navigation.PopAsync());
			page.Navigation.InsertPageBefore(advising, page);
			await page.Navigation.PopAsync();
		});
		return page;
	}

	private static ContentPage CreateReview(Page details)
	{
		var page = CreatePage("Registration review", out var layout);
		AddAction(page, layout, "Finish and remove prior detail", async () =>
		{
			page.Navigation.RemovePage(details);
			await page.Navigation.PopAsync();
		});
		AddAction(page, layout, "Return to root", async () =>
			await page.Navigation.PopToRootAsync());
		return page;
	}

	private static ContentPage CreateHelp()
	{
		var page = CreatePage("Campus help modal", out var layout);
		AddAction(page, layout, "Close", async () => await page.Navigation.PopModalAsync());
		return page;
	}

	private static ContentPage CreateStaticPage(string title)
	{
		return CreatePage(title, out _);
	}

	public static Page CreateNavigationRoot() => new NavigationPage(CreateCatalog());

	public static Page CreateTabbedRoot()
	{
		var tabs = new TabbedPage();
		tabs.Children.Add(new NavigationPage(CreateStaticPage("Home"))
			{ Title = "Home", IconImageSource = "home.png" });
		tabs.Children.Add(new NavigationPage(CreateCatalog())
			{ Title = "Courses", IconImageSource = "courses.png" });
		tabs.Children.Add(new NavigationPage(CreateStaticPage("Campus"))
			{ Title = "Campus", IconImageSource = "campus.png" });
		tabs.Children.Add(new NavigationPage(CreateStaticPage("Profile"))
			{ Title = "Profile", IconImageSource = "profile.png" });
		return tabs;
	}

	public static Page CreateFlyoutRoot()
	{
		var menu = CreatePage("University menu", out var layout);
		var flyout = new FlyoutPage
		{
			Flyout = menu,
			Detail = new NavigationPage(CreateCatalog())
		};
		AddAction(menu, layout, "Courses", () =>
		{
			flyout.Detail = new NavigationPage(CreateCatalog());
			flyout.IsPresented = false;
			return Task.CompletedTask;
		});
		AddAction(menu, layout, "Campus", () =>
		{
			flyout.Detail = new NavigationPage(CreateStaticPage("Campus services"));
			flyout.IsPresented = false;
			return Task.CompletedTask;
		});
		return flyout;
	}
}
```

The lab's local Clicked handlers use `async void` because they are UI event handlers, with exceptions caught inside them. The main assignment continues to use MVVM commands. The lab passes `courseId` through a method argument when it creates the detail page; a non-Shell page can likewise receive a model through its constructor. Shell query delivery is not involved.

### 12.3 Replace the lab's App.xaml.cs

```csharp
namespace UniversityNavigator.NavigationLab;

public partial class App : Application
{
	public App()
	{
		InitializeComponent();
	}

	protected override Window CreateWindow(IActivationState? activationState) =>
		new(NavigationLabPages.CreateNavigationRoot());
}
```

Select the lab project as the MAUI startup project, select your mobile target, and launch it. Do not launch the main app accidentally and conclude that the lab changes failed.

### 12.4 Perform the stack experiments

1. Open COMP301, then Back. `PushAsync` creates `[Catalog, Details]`; `PopAsync` returns to `[Catalog]`.
2. Open COMP301, then registration review. Use Return to root. `PopToRootAsync` removes all pushed pages from this NavigationPage stack.
3. Repeat, but use Finish and remove prior detail. `RemovePage(details)` removes the middle page while review remains current; popping review then returns directly to the catalog. Use this cautiously for completed wizard steps, not as the default Back implementation.
4. Open COMP301 and choose Insert advising before this page. `InsertPageBefore(advising, page)` changes `[Catalog, Details]` to `[Catalog, Advising, Details]`; popping Details reveals Advising.
5. From the catalog, open the help modal and Close. This time the pair is `PushModalAsync` and `PopModalAsync`, with no Shell route involved.

The navigation and modal stacks are distinct. `PopToRootAsync` does not mean "dismiss all modals." Do not remove the current page with RemovePage or try to remove the root. Prefer ordinary push/pop or a route reset when it expresses the workflow. These mutations are an advanced non-Shell comparison, not instructions to edit Shell's read-only Tab.Stack.

### 12.5 Compare alternative roots

For each experiment, replace only `CreateNavigationRoot()` in the lab's `CreateWindow` with the indicated factory, then relaunch:

| Factory | What to observe |
| --- | --- |
| `CreateTabbedRoot()` | Four TabbedPage children, each with its own NavigationPage. Open Courses details, switch tabs, and inspect retained history. Placement follows platform defaults. |
| `CreateFlyoutRoot()` | Menu/detail composition. Selecting a menu action replaces Detail with a new NavigationPage, intentionally discarding that previous detail history. |

NavigationPage inside TabbedPage or FlyoutPage is a non-Shell composition. It does not make either container compatible with Shell. A NavigationPage is required for the lab's modeless PushAsync calls; a bare ContentPage root does not provide the same navigation host.

Return the startup selection to `UniversityNavigator.App.csproj` when finished. The separate sandbox must not change the main app's four-tab structure.

## 13. Distinguish in-app routes from incoming deep links

Opening a course with Shell is **in-app navigation**. Receiving an address such as `northbridge://courses/COMP301` from an email is **OS-level app activation**. `Routing.RegisterRoute` alone does not register an Android intent filter, iOS URL scheme/universal link, or Windows protocol handler.

For this first navigation assignment, write the design and test the route mapping in-app; device-level registration is an extension, not a hidden prerequisite. A complete incoming-link implementation would:

1. Register the allowed scheme/host with each target platform. Verified HTTPS app links/universal links also require a controlled website and association files.
2. Receive the URI through platform activation/lifecycle handling, for both cold and warm starts.
3. Parse it with `Uri`, validate scheme, host, path shape, and allowed course ID, and reject unknown destinations. Never forward arbitrary external strings directly to `GoToAsync`.
4. Queue a valid request until the app window, Shell, route registrations, and any required sign-in are ready.
5. On the UI thread, select a known root and then push the validated detail; define whether an existing stack should be reset.

The final in-app mapping, once `courseId` is validated and Shell is ready, is:

```csharp
await Shell.Current.GoToAsync("//university/courses/catalog");
await Shell.Current.GoToAsync(
	$"course-details?courseId={Uri.EscapeDataString(courseId)}");
```

This is an illustrative handler body, not a standalone file: `courseId` comes from your validated activation input. Explain why an object reference from another process cannot be used as the payload, and why Back should land at the catalog after this entry path. Native .NET MAUI does not provide Xamarin.Forms-style `OnAppLinkRequestReceived` as a universal replacement for platform activation setup.

## 14. Verify the app with DevFlow

Compilation is necessary but insufficient. A passing build does not prove that a tab is visible, a constructor was activated correctly, a query reached its view model, or a native input has readable colors.

### 14.1 Use a repeatable inspection workflow

With the main app running under the MAUI debugger:

1. Wait for the agent using `maui_wait`, then identify the intended app with `maui_list_agents`. Select it if more than one app is registered.
2. Use `maui_capabilities` and `maui_list_actions`. No custom actions are required by this assignment; an empty actions list is acceptable.
3. Use `maui_tree` with a limited depth to inspect the page and controls. Verify visible elements have non-zero bounds.
4. Locate controls by the supplied AutomationIds. Tree element IDs can change after navigation or Hot Reload; query fresh IDs rather than reusing an old one.
5. Use `maui_tap`, `maui_fill`, and `maui_assert` to perform and check the journeys below. Native tab selection can also be performed manually if the platform tree does not expose a directly tappable tab element.
6. Query `TextColor`, `BackgroundColor`, and `PlaceholderColor` on StudentNameEntry with `maui_get_property`. A brush-backed Background or transparent managed property can require inspecting the parent and native rendering too.
7. Capture and inspect `maui_screenshot`. Native Entry styling can override the apparent XAML intent; a matching managed property alone does not prove the native surface looks correct.
8. Switch between light and dark themes and repeat input checks. Verify selected/unselected tabs, selected course rows, disabled actions, and text at increased device font size. On a smaller display, confirm the catalog has usable height and editor buttons remain reachable with the keyboard open.

When using a MAUI-enabled coding assistant, ask it to use these tools on the running app. Otherwise use the DevFlow inspector available through your installed tooling and record equivalent tree/property/screenshot evidence. Never submit an assistant's prediction as a screenshot or runtime assertion.

Useful stable assertions include:

| AutomationId | Property/value to verify |
| --- | --- |
| CourseTitle | Text equals the selected course title |
| InstructorName | Text equals that course's instructor |
| FavoriteResult | Text contains the chosen course ID |
| CampusHelpHeading | Visible on modal presentation, absent after dismissal |
| StudentName | Text changes after Save, not after Discard |
| StudentNameEntry | Text retains the draft after Stay |
| ProfileValidation | Non-empty after attempting to save a blank name |

Aim for WCAG AA contrast: at least 4.5:1 for normal text and 3:1 for large text. Record actual foreground/background colors, not just style resource names. If native rendering is low contrast, correct the source styles or platform handler as appropriate and verify again.

### 14.2 Required behavioral checklist

- [ ] Exactly four main tabs and four distinct tab icons on a mobile target.
- [ ] Home shortcut selects the Courses root without creating a Home-to-Courses Back chain.
- [ ] All three supplied courses open correctly by ID, object, and .NET 11 path.
- [ ] Missing/unknown course inputs produce a recoverable error, with invalid actions disabled.
- [ ] Course -> instructor -> course preserves the original course.
- [ ] Favorite result reaches the existing catalog page.
- [ ] One-level Back, two-level Back, and catalog reset have the recorded stack effects.
- [ ] Tab selection versus explicit root reset has been inspected, including retained per-tab history.
- [ ] Modal presentation/dismissal leaves the underlying tab intact.
- [ ] Profile Save, blank validation, Stay, and Discard all behave as specified.
- [ ] Toolbar Back, device Back/gesture, and tab-change guard behavior are recorded.
- [ ] Browser opens, or reports failure; missing email handlers do not crash the app.
- [ ] Repeated taps do not produce unintended duplicate detail pages.
- [ ] Light/dark native input contrast and small-screen/large-text layout are verified.
- [ ] Shell comparison changes are restored and the non-Shell sandbox is separate.

For each visual check, record both lines using your actual observations:

```text
Verified: [element] found at bounds (x, y, width, height), visible, with [expected value].
Accessibility: [control], TextColor=[hex], Background=[hex], PlaceholderColor=[hex or N/A]; [contrast result and screenshot reference].
```

If a check fails, describe the observed result and fix it before claiming that case passes. If your device is unavailable, say "not run" rather than substituting build output for runtime evidence.

## 15. Troubleshooting

| Symptom | Check first |
| --- | --- |
| Template rejects net11.0 | Run `dotnet --version` inside the solution folder; check global.json and MAUI template/workload compatibility. |
| Template runs, new app fails to activate a page | Check page and view-model DI registrations, constructor names, and x:Class. |
| Route not found | Compare the exact call with AppShell registrations; distinguish hierarchy roots from global detail routes. |
| Ambiguous/duplicate route | Remove conflicting registrations and unused experimental aliases; do not register visual routes a second time. |
| Path-parameter route fails | Confirm MAUI 11 Preview 7+ and use the absolute route with the catalog anchor. |
| Empty detail labels | Inspect incoming keys, BindingContext, and x:DataType. Do not create another view model in OnAppearing. |
| Detail returns with stale/reset data | Inspect repeated dictionary delivery and avoid resetting state on an empty parameter dictionary. |
| Property/command not found | Ensure the view model is partial, Toolkit package restored, and generated member spelling matches the binding. |
| Core model cannot resolve in XAML | Check the project reference and the Models mapping with `AssemblyName = "UniversityNavigator.Core"` in GlobalXamlNamespaces.cs. |
| Unprefixed page or view model cannot resolve in XAML | Check its exact CLR namespace is mapped in GlobalXamlNamespaces.cs in the app project; mappings do not include child namespaces automatically. |
| Tab icons missing | Verify SVG build inputs, lowercase filenames, PNG runtime references, and no duplicate resource base names. |
| Scrolling broken | CollectionView must occupy the Grid's star row, not a VerticalStackLayout or outer ScrollView. |
| Can't navigate after a prompt | Ensure every acquired deferral calls Complete in finally; never navigate while it is pending. |
| Wrong app launches | Select the main app, not the NavigationLab project, as startup. |
| DevFlow has no agent | Check debug launch integration and selected project/device; run extension diagnostics, not repeated package installs. |
| UI unchanged after editing | Try Hot Reload first; restart through the MAUI debugger only when changes cannot apply. |

If debugger launch reports a build error, obtain detailed output for the selected platform. For example, from the solution root for Android:

```console
dotnet build src/UniversityNavigator.App/UniversityNavigator.App.csproj -f net11.0-android -p:MauiDevFlowEnabled=true -bl:university-navigation.binlog
```

This is a diagnostic fallback, not a routine prerequisite to every debugger launch. Do not add `--no-restore`. Use your actual generated target framework for another platform. The extension-managed DevFlow package injection must be available to the build for its instrumentation to be present. Treat binlogs as potentially sensitive local diagnostics; do not include them in a public submission without review.

## 16. Submission and assessment

Submit the solution with the main app, Core library, and NavigationLab project, including global.json and project files. Exclude generated `bin` and `obj` directories, signing material, credentials, and device identifiers. Include a short report and a screen recording or screenshots showing the following:

1. Four mobile tabs with icons and the Home shortcut.
2. One complete course/instructor journey and returned favorite result.
3. Before/after stack diagrams for Back, multi-level Back, tab switching, root reset, and modal dismissal, supported by runtime observations.
4. A profile edit that is saved, a dirty edit that stays, and a dirty edit that is discarded.
5. Light/dark Entry property values and screenshots, including keyboard-visible layout.
6. The non-Shell sandbox's push/pop, pop-to-root, insert/remove, modal, tab, and flyout experiments.
7. Your SDK, MAUI workload, platform, and device/emulator type, plus any failed or unrun checks.

Answer these questions in your own words:

- Why is a tab root not the same as a pushed detail page?
- Why is `//course-details` invalid while the anchored .NET 11 template route is valid?
- When would you pass an ID instead of an object, and what happens after process termination?
- What differs between a regular dictionary and ShellNavigationQueryParameters?
- Why must query-string and path-parameter decoding be handled differently here?
- Why does Save return a new StudentProfile rather than mutate the parent's record while typing?
- Why does changing tabs not necessarily discard an editor or its history?
- Why must a navigation deferral complete even when the prompt fails?
- Why is registering a Shell route insufficient to receive an OS deep link?
- When would a standalone NavigationPage, TabbedPage, or FlyoutPage be an alternative to Shell?

| Area | Weight |
| --- | --- |
| Reproducible setup, references, MVVM, compiled bindings, and four-tab structure | 20% |
| Correct routes, detail data, instructor journey, and returned results | 25% |
| Stack reasoning, modal behavior, and unsaved-change protection | 25% |
| Comparison labs and explanations | 15% |
| Runtime/DevFlow evidence, accessibility, and error handling | 15% |

## References

- [.NET MAUI 11 changes and preview requirements](https://learn.microsoft.com/dotnet/maui/whats-new/dotnet-11?view=net-maui-11.0)
- [Shell navigation, route templates, data, and deferrals](https://learn.microsoft.com/dotnet/maui/fundamentals/shell/navigation?view=net-maui-11.0)
- [Shell tabs](https://learn.microsoft.com/dotnet/maui/fundamentals/shell/tabs?view=net-maui-11.0)
- [Shell flyout](https://learn.microsoft.com/dotnet/maui/fundamentals/shell/flyout?view=net-maui-11.0)
- [NavigationPage and modal navigation](https://learn.microsoft.com/dotnet/maui/user-interface/pages/navigationpage?view=net-maui-11.0)
- [TabbedPage](https://learn.microsoft.com/dotnet/maui/user-interface/pages/tabbedpage?view=net-maui-11.0)
- [FlyoutPage](https://learn.microsoft.com/dotnet/maui/user-interface/pages/flyoutpage?view=net-maui-11.0)
- [Dependency injection](https://learn.microsoft.com/dotnet/maui/fundamentals/dependency-injection?view=net-maui-11.0)
- [XAML namespaces and .NET 11 implicit declarations](https://learn.microsoft.com/dotnet/maui/xaml/namespaces?view=net-maui-11.0)
- [Mapping CLR namespaces with XmlnsDefinition](https://learn.microsoft.com/dotnet/maui/xaml/namespaces/custom-namespace-schemas?view=net-maui-11.0)
- [Compiled bindings and x:DataType](https://learn.microsoft.com/dotnet/maui/fundamentals/data-binding/compiled-bindings?view=net-maui-11.0)
- [Browser](https://learn.microsoft.com/dotnet/maui/platform-integration/appmodel/open-browser?view=net-maui-11.0)
- [Launcher](https://learn.microsoft.com/dotnet/maui/platform-integration/appmodel/launcher?view=net-maui-11.0)
- [CommunityToolkit.Mvvm](https://learn.microsoft.com/dotnet/communitytoolkit/mvvm/)

Instructor verification note: this document supplies a guided implementation and runtime test protocol. Markdown/XML checks do not establish that the complete sample has been built or run against your selected .NET 11 preview. Perform the mobile checkpoints with the course's pinned SDK/workload before distributing a claim of device-tested compatibility.
