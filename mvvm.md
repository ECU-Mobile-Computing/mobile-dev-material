# MVVM in .NET MAUI with CommunityToolkit.MVVM

This guide is for students who are brand new to XAML, C#, and the Model-View-ViewModel (MVVM) pattern. It focuses only on .NET 10+ MAUI and uses CommunityToolkit.Mvvm.

The goal is simple: learn how a MAUI screen is split into separate responsibilities so your app is easier to understand, test, and change.

## The big idea in one sentence

MVVM separates what the user sees (the View), what the app data is (the Model), and the logic that connects them (the ViewModel).

A common beginner-friendly mental model is:

- View: the screen and UI controls
- ViewModel: the screen logic and data for that screen
- Model: the data itself

## Key components at a glance

These are the main pieces you need to understand:

- Model
- View
- ViewModel
- Binding
- Command
- ObservableObject
- ObservableProperty
- RelayCommand
- INotifyPropertyChanged
- Async and validation patterns
- Common MVVM mistakes and best practices

Now let’s look at each one.

## 1. Model

The Model represents the data your app uses.

It is usually a simple C# class. It does not know about the screen or UI controls. It only describes the data.

Example:

```csharp
namespace MyApp.Models;

public class Person
{
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
}
```

This is data, not UI logic. The Model can contain business data, such as:

- a customer
- a product
- a task
- a user profile
- a weather object

A common beginner mistake is putting UI code into the Model. The Model should stay focused on data.

## 2. View

The View is the XAML screen. It defines what the user sees: buttons, labels, text boxes, lists, and layout.

In MAUI, the View is usually a `.xaml` file plus a `.xaml.cs` code-behind file.

Example view:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage x:Class="MyApp.MainPage">

    <VerticalStackLayout Padding="24" Spacing="12">
        <Entry Placeholder="Type your name" />
        <Button Text="Greet" />
        <Label Text="Hello!" FontSize="24" />
    </VerticalStackLayout>
</ContentPage>
```

Simple XAML primer:

- `ContentPage` is a screen
- `VerticalStackLayout` arranges children vertically
- `Entry` is a text box
- `Button` is a clickable button
- `Label` shows text

XAML is just a structured way to define UI objects.

A View is not supposed to do too much business logic. It should mostly display data and send user actions upward to the ViewModel.

## 3. ViewModel

The ViewModel is the bridge between the View and the Model.

It holds the data that the View should display and contains the logic for what happens when the user interacts with the screen.

A ViewModel usually:

- exposes properties for the UI
- contains commands for button clicks
- transforms data for display
- responds to user actions

Example ViewModel:

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace MyApp.ViewModels;

public partial class MainViewModel : ObservableObject
{
    [ObservableProperty]
    private string name = string.Empty;

    [ObservableProperty]
    private string greeting = "Hello!";

    [RelayCommand]
    private void Greet()
    {
        if (string.IsNullOrWhiteSpace(Name))
        {
            Greeting = "Please enter your name.";
            return;
        }

        Greeting = $"Hello, {Name}!";
    }
}
```

This is a very common beginner MVVM pattern in MAUI.

## 4. Binding

Binding connects the UI to the ViewModel.

Instead of writing code to manually update every label, you bind a UI element to a property.

Example in XAML:

```xml
<Entry Text="{Binding Name}" Placeholder="Type your name" />
<Label Text="{Binding Greeting}" FontSize="24" />
<Button Text="Greet" Command="{Binding GreetCommand}" />
```

This says:

- the `Entry` text is bound to the `Name` property
- the `Label` text is bound to the `Greeting` property
- the `Button` click is bound to the `GreetCommand`

The binding source is the `BindingContext`.

In the page code-behind:

```csharp
using MyApp.ViewModels;

public partial class MainPage : ContentPage
{
    public MainPage()
    {
        InitializeComponent();
        BindingContext = new MainViewModel();
    }
}
```

This is the key: the View binds to the ViewModel, and the ViewModel raises notifications when data changes.

## 5. Commands

A Command is the MVVM way to handle button clicks and other actions.

In MAUI, a button can execute a command instead of using a click event in code-behind.

The command pattern is useful because it keeps UI actions in the ViewModel, not in the page.

Example:

```csharp
[RelayCommand]
private void Greet()
{
    Greeting = $"Hello, {Name}!";
}
```

This generates a command named `GreetCommand` for the button.

The button uses it like this:

```xml
<Button Text="Greet" Command="{Binding GreetCommand}" />
```

This is cleaner and more testable than writing button click handlers directly in the page.

## 6. ObservableObject

`ObservableObject` is the base class from CommunityToolkit.Mvvm that helps the UI know when a property changed.

It supports `INotifyPropertyChanged` for you, which is something that the UI listens to.

When a property changes, the UI updates automatically.

Example:

```csharp
using CommunityToolkit.Mvvm.ComponentModel;

public partial class MainViewModel : ObservableObject
{
    [ObservableProperty]
    private string name = string.Empty;
}
```

The `[ObservableProperty]` attribute tells the toolkit to generate the property and raise change notifications.

## 7. ObservableProperty

`[ObservableProperty]` is one of the most important CommunityToolkit.Mvvm features for beginners.

Instead of manually writing:

```csharp
private string name;

public string Name
{
    get => name;
    set => SetProperty(ref name, value);
}
```

you write:

```csharp
[ObservableProperty]
private string name = string.Empty;
```

The toolkit generates the `Name` property and the change notification automatically.

This keeps the code shorter and easier to read.

## 8. RelayCommand

`[RelayCommand]` generates a command for you.

This is the MVVM-friendly way to handle actions from the View.

Example:

```csharp
[RelayCommand]
private void Save()
{
    // do work here
}
```

This generates a command property named `SaveCommand` that can be used in XAML.

```xml
<Button Text="Save" Command="{Binding SaveCommand}" />
```

For async methods, you can also use:

```csharp
[RelayCommand]
private async Task LoadDataAsync()
{
    await Task.Delay(1000);
}
```

This generates `LoadDataAsyncCommand`.

## 9. INotifyPropertyChanged

This is the interface behind most of MVVM in .NET.

When a property changes, the UI needs to know so it can refresh its display. `ObservableObject` handles this for you.

Without this, the UI may not update when data changes.

So, in plain English:

- the ViewModel changes a value
- `ObservableObject` raises a property-changed event
- the UI sees the event and updates the label, text box, or list

That is why binding works so well in MVVM.

## 10. The full beginner example

Here is a complete minimal example showing the pattern clearly.

### Model

```csharp
namespace MyApp.Models;

public class GreetingModel
{
    public string Name { get; set; } = string.Empty;
}
```

### ViewModel

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

namespace MyApp.ViewModels;

public partial class GreetingViewModel : ObservableObject
{
    [ObservableProperty]
    private string name = string.Empty;

    [ObservableProperty]
    private string greetingText = "Hello!";

    [RelayCommand]
    private void UpdateGreeting()
    {
        if (string.IsNullOrWhiteSpace(Name))
        {
            GreetingText = "Please enter your name.";
            return;
        }

        GreetingText = $"Welcome, {Name}!";
    }
}
```

### View

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage x:Class="MyApp.MainPage">

    <VerticalStackLayout Padding="24" Spacing="12">
        <Entry Text="{Binding Name}" Placeholder="Type your name" />
        <Button Text="Update greeting" Command="{Binding UpdateGreetingCommand}" />
        <Label Text="{Binding GreetingText}" FontSize="24" />
    </VerticalStackLayout>
</ContentPage>
```

### Code-behind

```csharp
using MyApp.ViewModels;

public partial class MainPage : ContentPage
{
    public MainPage()
    {
        InitializeComponent();
        BindingContext = new GreetingViewModel();
    }
}
```

This example is intentionally small, but it demonstrates the full flow:

1. User types text in the `Entry`
2. The `Name` property updates
3. The user presses the button
4. `UpdateGreetingCommand` runs
5. The `GreetingText` property updates
6. The `Label` refreshes automatically through binding

## 11. Async and validation patterns

Real apps often need to do work asynchronously, such as loading data from a database or API.

A common MVVM pattern is to use `Task` in the ViewModel:

```csharp
[RelayCommand]
private async Task LoadDataAsync()
{
    IsBusy = true;

    try
    {
        await Task.Delay(1000);
        GreetingText = "Data loaded.";
    }
    finally
    {
        IsBusy = false;
    }
}
```

You may also want properties like `IsBusy` and `HasError` to inform the UI while a task is running.

```csharp
[ObservableProperty]
private bool isBusy;

[ObservableProperty]
private string errorMessage = string.Empty;
```

This is a clean, beginner-friendly way to manage app state.

## 12. Common beginner mistakes

Here are a few issues that students often run into:

### Mistake 1: Putting UI logic in the page code-behind

This works for a tiny app, but it becomes messy quickly. In MVVM, the page mostly binds to data and commands.

### Mistake 2: Binding directly to the wrong object

If the page has no `BindingContext`, the UI will not update. Make sure the page sets the `BindingContext` to the ViewModel.

### Mistake 3: Forgetting to use `ObservableProperty`

If you change a property without notifying the UI, the screen may not refresh.

### Mistake 4: Putting business logic in the View

The View should not know how to validate or process app logic. That belongs in the ViewModel.

### Mistake 5: Making the ViewModel too big

If the ViewModel becomes crowded, split it up. A screen-specific ViewModel should stay focused.

## 13. Best practices for .NET 10+ MAUI MVVM

Use these habits as you build apps:

- Keep the View thin
- Keep the ViewModel focused on page logic and state
- Keep the Model focused on data
- Use `ObservableObject` and `[ObservableProperty]`
- Use `[RelayCommand]` for button actions
- Bind in XAML instead of updating controls from code-behind
- Keep logic testable and easy to understand
- Avoid business rules in the page
- Use `async` and `Task` for background operations

## 14. A simple rule to remember

If you are unsure where code belongs, ask this:

- UI display? Put it in the View.
- Screen behavior and state? Put it in the ViewModel.
- Data itself? Put it in the Model.

That simple rule will help you start most MAUI MVVM apps correctly.

## 15. Final takeaway

MVVM is not magic. It is a way to organize code so that:

- the UI is easier to read
- logic is easier to test
- state changes are easier to manage
- apps stay maintainable as they grow

For beginners, the best first step is to learn this pattern with small screens:

- one View
- one ViewModel
- one Model
- bound properties
- commands for actions

Once you understand that flow, your MAUI apps become much easier to build and extend.

## Quick checklist

Before you finish a screen, ask yourself:

- Is the View mostly markup and binding?
- Is the ViewModel holding screen state and behavior?
- Are the properties using `ObservableProperty`?
- Are the actions using `[RelayCommand]`?
- Is the Model just data, not UI logic?
- Are labels and text boxes updated by binding, not manual code?

If the answer is yes, you are following the MVVM pattern well.

## Summary

The most important parts of MVVM in MAUI are:

- Model: the data
- View: the screen
- ViewModel: the screen logic
- Binding: the connection between View and ViewModel
- ObservableObject: property change notifications
- ObservableProperty: generated properties
- RelayCommand: generated button actions
- INotifyPropertyChanged: the notification system behind the scenes

This is the foundation of modern .NET MAUI app architecture, and it is a very good pattern for new developers to learn.
