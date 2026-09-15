# MAUI Shell Navigation Cheat Sheet

This cheat sheet is a quick reference for navigating between pages in a .NET MAUI app using Shell. It starts with the easiest patterns and moves toward more advanced Shell navigation concepts.

## 1. What is Shell navigation?

MAUI Shell gives you a modern way to organize an app as a hierarchy of routes, tabs, and flyout items. Instead of manually managing a stack of pages, you can navigate using route-based commands.

Common Shell navigation tools:

- `Shell.Current.GoToAsync(...)` - navigate to a route
- `Shell.Current.Navigation.PopAsync()` - go back one page
- `Shell.Current.Navigation.PopToRootAsync()` - return to the root page
- `Shell.Current.GoToAsync("..")` - go back one level in a relative route
- `Shell.Current.CurrentState.Location` - inspect the current route

## 2. Basic app shell structure

A Shell app usually has one `AppShell` file that defines the app structure.

```xml
<Shell x:Class="MyApp.AppShell"
      xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
      xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
      xmlns:local="clr-namespace:MyApp">

    <TabBar>
        <Tab Title="Home" Route="home">
            <ShellContent Title="Home"
                          Route="main"
                          ContentTemplate="{DataTemplate local:HomePage}" />
        </Tab>

        <Tab Title="Settings" Route="settings">
            <ShellContent Title="Settings"
                          Route="main"
                          ContentTemplate="{DataTemplate local:SettingsPage}" />
        </Tab>
    </TabBar>
</Shell>
```

This creates a Shell with two tabs:

- `home/main`
- `settings/main`

The route path matters when you navigate between pages.

## 3. Simple navigation

The simplest Shell navigation pattern is to move to a page by route.

### Example: open a details page

```csharp
await Shell.Current.GoToAsync(nameof(DetailsPage));
```

If your route is registered by the page, this is the most common beginner pattern.

### Example with route registration

In `AppShell.xaml.cs`:

```csharp
public partial class AppShell : Shell
{
    public AppShell()
    {
        InitializeComponent();
        Routing.RegisterRoute(nameof(DetailsPage), typeof(DetailsPage));
    }
}
```

Then navigate:

```csharp
await Shell.Current.GoToAsync(nameof(DetailsPage));
```

This is simple, readable, and works well for many apps.

### Example with a page route explicitly defined

```xml
<ContentPage x:Class="MyApp.DetailsPage"
             xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             Title="Details"
             Shell.PresentationMode="Animated">
    <VerticalStackLayout Padding="20">
        <Label Text="Details page" FontSize="24" />
    </VerticalStackLayout>
</ContentPage>
```

Then you can navigate to it with:

```csharp
await Shell.Current.GoToAsync("details");
```

This works if the route is configured as `details` in the Shell hierarchy.

## 4. Passing data with query strings

Shell supports query-string style parameters in routes.

### Example

```csharp
await Shell.Current.GoToAsync($"details?itemId={itemId}&name={name}");
```

In the target page, read the values:

```csharp
public partial class DetailsPage : ContentPage
{
    public DetailsPage()
    {
        InitializeComponent();
    }

    protected override async void OnAppearing()
    {
        base.OnAppearing();

        var itemId = Shell.Current.CurrentState.Location.Query["itemId"];
        var name = Shell.Current.CurrentState.Location.Query["name"];

        await DisplayAlert("Details", $"Item: {itemId}, Name: {name}", "OK");
    }
}
```

The query string is a common way to pass simple values without needing a custom parameter object.

### Better pattern for a route that expects a parameter

```csharp
await Shell.Current.GoToAsync($"product/{productId}");
```

This is a route with a path segment instead of a query string.

```csharp
Routing.RegisterRoute("product/{productId}", typeof(ProductPage));
```

Then in `ProductPage`:

```csharp
public ProductPage()
{
    InitializeComponent();
}

protected override void OnAppearing()
{
    base.OnAppearing();

    var productId = Shell.Current.CurrentState.Location.ToString();
}
```

For most beginner projects, query strings are easier to understand than custom route parameter patterns.

## 5. Back navigation

Back tracking is one of the most important Shell concepts.

### Go back one page with a relative route

```csharp
await Shell.Current.GoToAsync("..");
```

This means: go back one level in the current route hierarchy.

### Go back using the navigation API

```csharp
await Shell.Current.Navigation.PopAsync();
```

This is the standard stack-based way to move backward.

### Go to the root page

```csharp
await Shell.Current.Navigation.PopToRootAsync();
```

This clears the stack and returns to the root of the navigation hierarchy.

### Go back to a specific absolute route

```csharp
await Shell.Current.GoToAsync("//home");
```

The `//` prefix means “start from the root Shell route.”

Example:

```csharp
await Shell.Current.GoToAsync("//settings");
```

This jumps directly to the settings tab or route, regardless of the current page stack.

## 6. Navigation stack management

Shell still uses a navigation stack behind the scenes. You can inspect or manipulate it.

### Check the current stack

```csharp
var stack = Shell.Current.Navigation.NavigationStack;

foreach (var page in stack)
{
    Console.WriteLine(page.GetType().Name);
}
```

You can also check the count:

```csharp
var count = Shell.Current.Navigation.NavigationStack.Count;
```

### Remove the current page from the stack

```csharp
var currentPage = Shell.Current.CurrentPage;
Shell.Current.Navigation.RemovePage(currentPage);
```

### Insert a page before the current one

```csharp
var page = new ProfilePage();
Shell.Current.Navigation.InsertPageBefore(page, Shell.Current.CurrentPage);
```

### Pop to the root

```csharp
await Shell.Current.Navigation.PopToRootAsync();
```

This is useful when a user logs out or resets the app flow.

## 7. Relative vs absolute routes

This distinction is very important in Shell.

### Relative route

A relative route is based on the current route.

```csharp
await Shell.Current.GoToAsync("details");
await Shell.Current.GoToAsync("..");
```

Relative routes are often used when you are already inside a Shell section.

### Absolute route

An absolute route starts from the root Shell.

```csharp
await Shell.Current.GoToAsync("//home");
await Shell.Current.GoToAsync("//settings");
```

Absolute routes are helpful when you want to jump directly to a tab or root page, even if the current page is nested deeply.

## 8. Advanced Shell features

### Flyout navigation

```xml
<Shell x:Class="MyApp.AppShell"
      xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
      xmlns:local="clr-namespace:MyApp">

    <FlyoutItem Title="Main">
        <ShellContent Title="Home" ContentTemplate="{DataTemplate local:HomePage}" Route="home" />
    </FlyoutItem>

    <FlyoutItem Title="Profile">
        <ShellContent Title="Profile" ContentTemplate="{DataTemplate local:ProfilePage}" Route="profile" />
    </FlyoutItem>
</Shell>
```

To navigate to a flyout page:

```csharp
await Shell.Current.GoToAsync("//profile");
```

### Tab-based navigation

```xml
<TabBar>
    <Tab Title="Home" Route="home">
        <ShellContent ContentTemplate="{DataTemplate local:HomePage}" />
    </Tab>

    <Tab Title="Search" Route="search">
        <ShellContent ContentTemplate="{DataTemplate local:SearchPage}" />
    </Tab>
</TabBar>
```

Navigate to a tab:

```csharp
await Shell.Current.GoToAsync("//search");
```

### Modal pages

A modal page is often shown over the current page.

```csharp
await Shell.Current.GoToAsync("modalpage");
```

When a page has a modal route, it may appear on top of the current UI and must be dismissed with `GoToAsync("..")` or `PopAsync`.

## 9. Common patterns in real apps

### Pattern 1: Simple page-to-page navigation

```csharp
private async void OnOpenDetailsClicked(object sender, EventArgs e)
{
    await Shell.Current.GoToAsync(nameof(DetailsPage));
}
```

### Pattern 2: Navigate with data

```csharp
private async void OnOpenProductClicked(object sender, EventArgs e)
{
    var productId = 42;
    await Shell.Current.GoToAsync($"product/{productId}");
}
```

### Pattern 3: Go back

```csharp
private async void OnBackClicked(object sender, EventArgs e)
{
    await Shell.Current.GoToAsync("..");
}
```

### Pattern 4: Return to root

```csharp
private async void OnLogoutClicked(object sender, EventArgs e)
{
    await Shell.Current.Navigation.PopToRootAsync();
    await Shell.Current.GoToAsync("//login");
}
```

## 10. Quick reference

### Best beginner commands

```csharp
await Shell.Current.GoToAsync(nameof(DetailsPage));
await Shell.Current.GoToAsync("details");
await Shell.Current.GoToAsync("..");
await Shell.Current.GoToAsync("//home");
await Shell.Current.Navigation.PopAsync();
await Shell.Current.Navigation.PopToRootAsync();
```

### Useful reminders

- Use `GoToAsync` for Shell-based navigation.
- Use `..` to go back one level.
- Use `//` to jump to root-level routes.
- Use `PopAsync` when you want classic stack behavior.
- Use `PopToRootAsync` when you want to reset the user flow.
- Use `NavigationStack` to inspect the current page stack.

## 11. Recommended beginner mindset

Start simple:

1. Define a `Shell` and routes.
2. Navigate by `GoToAsync(nameof(Page))`.
3. Pass simple values with query strings.
4. Use `..` and `PopAsync` for back navigation.
5. Use `//root` and `PopToRootAsync` for full reset or route jumps.

As your app grows, Shell becomes extremely helpful because it organizes navigation around routes, tabs, flyout items, and page stacks instead of a custom manual navigation system.

## 12. One last example: full flow

```csharp
public partial class HomePage : ContentPage
{
    public HomePage()
    {
        InitializeComponent();
    }

    private async void OnViewItemClicked(object sender, EventArgs e)
    {
        var itemId = 7;
        var name = "Laptop";

        await Shell.Current.GoToAsync($"details?itemId={itemId}&name={name}");
    }
}
```

Then in `DetailsPage`:

```csharp
private async void OnBackClicked(object sender, EventArgs e)
{
    await Shell.Current.GoToAsync("..");
}
```

This pattern is very common in MAUI apps: open a detail page, inspect the route or query data, and then go back with `..`.

## Summary

The key idea is simple:

- `GoToAsync` moves forward through routes.
- `..` moves back relative to the current page.
- `PopAsync` moves back through the navigation stack.
- `PopToRootAsync` resets to the root.
- `//` jumps to an absolute route in the Shell.

Use these patterns together, and your MAUI app will feel much easier to navigate and maintain.
