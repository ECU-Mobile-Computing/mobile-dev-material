# MAUI XAML Cheat Sheet for Beginners (.NET 11)

This cheat sheet is designed for students who have never used XML or XAML before. The goal is to teach the XAML part of MAUI without extra distractions. It focuses on the patterns you will see in a typical .NET 11 MAUI app.

Important: after the namespace section, most examples below use the MAUI global/default XAML namespace. That means the controls are used implicitly, without a prefix. This is the most common beginner pattern in MAUI XAML.

## 1. What is XAML?

XAML stands for Extensible Application Markup Language. It is a declarative way to describe a UI using XML-like syntax.

- XAML declares objects and their properties.
- A MAUI control like `Label` or `Button` is just an object.
- A property like `Text` or `BackgroundColor` is assigned with an attribute.
- XAML often works together with C# code-behind.

Example:

```xml

<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui">
    <VerticalStackLayout>
        <Label Text="Hello, XAML!" />
        <Button Text="Tap me" />
    </VerticalStackLayout>
</ContentPage>
```

### Mental model

Think of XAML like this:

- `ContentPage` = a page object
- `VerticalStackLayout` = a layout container object
- `Label` = a visual object
- `Text="Hello"` = a property assignment

## 2. XML basics you need to know

XAML is XML-based, so a few rules matter:

- Elements are written with angle brackets: `<Label />`
- Elements can have attributes: `Text="Hello"`
- Child elements go inside parent elements
- Tags must be closed (either with `/>` or a closing tag)
- Attribute values are strings or special XAML values

Example:

```xml
<Label Text="My label" FontSize="18" />
```

## 3. Namespaces and prefixes

A namespace tells XAML which set of elements you are using. In MAUI, the most important namespace is the MAUI framework namespace.

### Prior to .Net 11

Before .Net 11, all root XAML elements in XAML files were required to include both the XAML and MAUI XML Namespaces.  In addition, the XML NameSpaces of all items referenced in the XAML file from another namespace were required to be added to the root XAML element.  Essentially, every item used that was not from an existing XML Namespace required another Namespace to be added to the XAML root element.

Below is what a typical XAML file used to look like.  x:, local:, components:, and vm: are all prefixes defined for their respective namespaces. x: is a reserved prefix for the XAML.  :local, :components, and vm: are all arbitrary namespaces that could be named just about anything, but the prefixes should be meaningful.  e.g. :components represents the namespace that contains several custom XAML components.

```xml
<ContentPage
    xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
    xmlns:local="clr-namespace:MyApp"
    xmlns:components="clr-namespace:MyApp.Components"
    xmlns:vm="clr-namespace:MyApp.ViewModels"
    x:Class="MyApp.MainPage"
    x:DataType="vm:MainPageViewModel">
    
    <Label Text="{Binding Title}" />

    <components:ToggleSwitch>
        Power Switch
    </components:ToggleSwitch>

    <local:InfoCard>
        <local:Subject>
            Task 1
        </local:Subject>
        <local:Body>
            Send an email.
        </local:Body>
    </local:InfoCard>

</ContentPage>
```

Using .Net 11's Implicit Usings and GlobalXmlns, the same code looks like this.  Only the actualy designation of the code behind (x:Class) and the view model (x:DataType) are required.  x: is the only prefix required.
```xml
<ContentPage
    x:Class="MyApp.MainPage"
    x:DataType="MainPageViewModel">
    
    <Label Text="{Binding Title}" />

    <ToggleSwitch>
        Power Switch
    </ToggleSwitch>

    <InfoCard>
        <Subject>
            Task 1
        </Subject>
        <Body>
            Send an email.
        </Body>
    </InfoCard>

</ContentPage>
```

### Default Namespace

This is the namespace you declare without a prefix (`xmlns=`) on the root element.  Only one namespace can be the default namespace. Prior to .Net 11, all unprefixed XAML elements were implicitly assigned the default namespace. 

### Implicit Usings
Implicit Usings eliminate the need to specify the MAUI and XAML XML Namespaces.

To enable, add the following XML element with the content of `enable`
```xml
<ImplicitUsings>enable</ImplicitUsings>
```

### Global XML Namespace

The concept of a global namespace was added in .Net 11.  If a namespace was added to GlobalXmlns.cs, the namespace would automatically be available to all other XAML files. This means that prefixes are not required for these namespaces. 

Below is the MAUI global namespace
```xml
"http://schemas.microsoft.com/dotnet/maui/global"
```

Currently, GlobalXmlns is a preview feature.  You must enable it in your MAUI project file.

Add the following XML elements to the <PropertyGroup> section of your MAUI projects .csproj file.
```xml
<LangVersion>preview</LangVersion>
<EnablePreviewFeatures>true</EnablePreviewFeatures>
```

Now we can add XML Namespaces to the Global Namespace and drop prefixes (all prefixes except x:).


You add a namespace to the global namespace included namespace in the GlobalXmlns.cs file like this
```xml
[assembly: XmlnsDefinition("http://schemas.microsoft.com/dotnet/maui/global", "MyApp.Components")]
[assembly: XmlnsDefinition("http://schemas.microsoft.com/dotnet/maui/global", "MyApp.ViewModels")]
[assembly: XmlnsDefinition("http://schemas.microsoft.com/dotnet/maui/global", "MyApp.Utilities")]
```


```xml
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui">
    <Label Text="Welcome" />
</ContentPage>
```

The MAUI namespace is the global namespace for most controls.

### x prefix

The `x:` prefix is for XAML directives, not for MAUI controls. It is used for special XAML features like:

- `x:Class` - specify the XAML file's code behind class
- `x:Name`  - give XAML element a name so you can refer to it in C# code
- `x:Static` - reference various static, const, and enum values
- `x:Reference` - reference another XAML control
- `x:DataType` - specify the view model that will be used for data binding

Example:

```xml
<ContentPage
    x:Class="MyApp.MainPage">

    <Label x:Name="welcomeLabel" Text="Hello" />
</ContentPage>
```

### Custom namespaces

Before .Net 11 and Global XML Namespace, you had to add a custom prefix such as `local` or `vm` when use your own classes, view models, or custom controls.  This is no longer necessary if you enable Global XML Namespaces.

```xml
<ContentPage
    xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
    xmlns:local="clr-namespace:MyApp"
    xmlns:vm="clr-namespace:MyApp.ViewModels"
    x:Class="MyApp.MainPage">

    
    <Label Text="Custom namespace example" />
</ContentPage>
```

## 4. Setting page content

Every `ContentPage` has a `Content` property. The page usually contains a single visual root element as its content.

Example:

```xml

<ContentPage x:Class="MyApp.HomePage">
    <VerticalStackLayout Padding="20" Spacing="12">
        <Label Text="Welcome to MAUI" />
        <Entry Placeholder="Type here" />
        <Button Text="Submit" />
    </VerticalStackLayout>
</ContentPage>
```

The single child inside `ContentPage` is the page content.

## 5. x:Class and the code-behind class

`x:Class` tells XAML which .NET class this page belongs to. The class must exist in code-behind and usually is a `partial` class.

```xml
<ContentPage x:Class="MyApp.MainPage">
    <Label Text="Hello from XAML" />
</ContentPage>
```

```csharp
namespace MyApp;

public partial class MainPage : ContentPage
{
    public MainPage()
    {
        InitializeComponent();
    }
}
```

### Relationship between XAML and code-behind

- XAML describes the UI
- C# contains the behavior
- `x:Class` links the two together
- `InitializeComponent()` runs the XAML loading logic

## 6. x:Name and code-behind interaction

`x:Name` creates a named field for a control so you can reference it in C#.

```xml
<ContentPage x:Class="MyApp.MainPage">
    <VerticalStackLayout>
        <Entry x:Name="nameEntry" Placeholder="Name" />
        <Button x:Name="saveButton" Text="Save" />
    </VerticalStackLayout>
</ContentPage>
```

```csharp
public partial class MainPage : ContentPage
{
    public MainPage()
    {
        InitializeComponent();
        nameEntry.Text = "Ada";
    }
}
```

### Why this matters

XAML controls can be accessed from code-behind by name after `InitializeComponent()`.

## 7. XAML and C# together

A common MAUI pattern is:

- XAML declares the UI
- C# handles events and logic

Example:

```xml
<ContentPage x:Class="MyApp.MainPage">
    <VerticalStackLayout>
        <Button Text="Show message" Clicked="ShowMessage_Clicked" />
    </VerticalStackLayout>
</ContentPage>
```

```csharp
public partial class MainPage : ContentPage
{
    public MainPage()
    {
        InitializeComponent();
    }

    private async void ShowMessage_Clicked(object sender, EventArgs e)
    {
        await DisplayAlert("Hello", "This came from C#!", "OK");
    }
}
```

## 8. Properties

Most XAML attributes set a property on an object.

```xml
<Label Text="Hello" FontSize="22" TextColor="DarkBlue" />
```

### Common property patterns

- `Text` sets text content
- `FontSize` sets size
- `BackgroundColor` sets a color
- `IsVisible` sets visibility
- `Padding` sets spacing around content

Example:

```xml

<Button
    Text="Click me"
    BackgroundColor="DodgerBlue"
    TextColor="White"
    Padding="12,8"
    CornerRadius="10" />
```

## 9. Property elements

Some properties are easier to write as child elements instead of attributes. This is called a property element.

Example:

```xml
<Label Text="A larger label">
    <Label.FontSize>24</Label.FontSize>
</Label>
```

This is equivalent to:

```xml
<Label Text="A larger label" FontSize="24" />
```

Use property elements when the value is more complex or more readable as nested content.

## 10. Attached properties

An attached property is a property set by a parent or container, not by the object itself. These are common in layouts.

Example:

```xml
<Grid Padding="20">
    <Label Text="Top left" Grid.Row="0" Grid.Column="0" />
    <Label Text="Top right" Grid.Row="0" Grid.Column="1" />
</Grid>
```

Label does not have `Grid.Row` and `Grid.Column` properties. These properties are defined by the Grid and known as Attached Properties. You can attach these properties to the children of the Grid to position its children inside the Grid. 

## 11. Content property

Some controls define a special content property. This means you can place content directly inside the element without naming the property.

Example:

```xml

<ContentPage x:Class="MyApp.HomePageg">
    <Label Text="This is page content" />
</ContentPage>
```

The label is the content of the page because `ContentPage` has a content property.

## 12. Built-in MAUI controls by category

The following sections list common built-in MAUI controls. They are all used with the global/default MAUI namespace unless a custom namespace is added.

### 12.1 Layout controls

These control how children are arranged.

- `Grid` — rows and columns
- `VerticalStackLayout` — stacks children vertically
- `HorizontalStackLayout` — stacks children horizontally
- `Border` — wraps content with a border
- `ScrollView` — lets content scroll
- `ContentView` — simple container
- `FlexLayout` — flexible layout model

Example:

```xml
<VerticalStackLayout Padding="20" Spacing="12">
    <Label Text="Name" />
    <Entry Placeholder="Enter name" />
    <Button Text="Save" />
</VerticalStackLayout>
```

### 12.2 Display controls

- `Label` — text
- `Image` — image display
- `BoxView` — simple colored box
- `WebView` — embed web content
- `GraphicsView` — custom drawing

Example:

```xml
<StackLayout>
    <Label Text="Student portal" FontSize="24" />
    <Image Source="dotnet_bot.png" HeightRequest="100" WidthRequest="100" />
</StackLayout>
```

### 12.3 Input controls
- `Entry` — single-line text input
- `Editor` — multi-line text input
- `Button` — clickable action
- `Checkbox` — toggle
- `Picker` — select from options
- `DatePicker` — choose a date
- `TimePicker` — choose a time
- `SearchBar` — search input
- `Slider` — continuous value input
- `Stepper` — increment/decrement values
- `Switch` — on/off toggle
- `RadioButton` — single-choice option

Example:

```xml
<VerticalStackLayout Padding="20" Spacing="10">
    <Entry Placeholder="Email" />
    <Editor Placeholder="Notes" AutoSize="TextChanges" />
    <CheckBox IsChecked="true" />
    <Picker Title="Choose a subject" />
    <Button Text="Submit" />
</VerticalStackLayout>
```

### 12.4 Collection and data controls

- `CollectionView` — flexible list/grid UI
- `CarouselView` — horizontal item carousel
- `RefreshView` — pull-to-refresh container

Example:

In the example below, Students is a list of Student objects.  Student object has a Name property. The ItemTemplate will be rendered for each Student in the Students list.

```xml
<CollectionView ItemsSource="{Binding Students}">
    <CollectionView.ItemTemplate>
        <DataTemplate>
            <Border Padding="12">
                <Label Text="{Binding Name}" />
            </Border>
        </DataTemplate>
    </CollectionView.ItemTemplate>
</CollectionView>
```

### 12.5 Navigation and page controls

- `ContentPage` — basic page
- `NavigationPage` — navigation container
- `TabbedPage` — tabs
- `FlyoutPage` — flyout menu pattern
- `Shell` — modern app shell navigation

Example:

```xml
<Shell>
    <TabBar>
        <Tab Title="Home">
            <ShellContent Title="Home" ContentTemplate="{DataTemplate homePage}" />
        </Tab>
    </TabBar>
</Shell>
```

## 13. Shared resources and styles

You can define reusable values like colors, brushes, and styles in `Application.Resources` or page resources.

Example:

```xml
<Application>
    <Application.Resources>
        <Color x:Key="PrimaryColor">#512BD4</Color>

        <Style x:Key="PrimaryButtonStyle" TargetType="Button">
            <Setter Property="BackgroundColor" Value="{StaticResource PrimaryColor}" />
            <Setter Property="TextColor" Value="White" />
        </Style>
    </Application.Resources>
</Application>
```

Then use it:

```xml
<Button Text="Save" Style="{StaticResource PrimaryButtonStyle}" />
```

### Shared resources in a page

```xml
<ContentPage x:Class="MyApp.MainPage">
    <ContentPage.Resources>
        <Color x:Key="AccentColor">#FF5722</Color>
    </ContentPage.Resources>

    <Label Text="Accent text" TextColor="{StaticResource AccentColor}" />
</ContentPage>
```

## 15. Markup extensions

Markup extensions are special XAML syntax that evaluate to values at runtime. MAUI heavily uses them.

### 15.1 StaticResource

Use this when you want to read a resource once.

```xml

<Label TextColor="{StaticResource AccentColor}" />
```

### 15.2 DynamicResource

Use this when the value should update if the resource changes.

```xml

<Label TextColor="{DynamicResource AccentColor}" />
```

### 15.3 x:Static

Use this to access static fields, properties, and constants.

```xml
<ContentPage x:Class="MyApp.MainPage"
             xmlns:sys="clr-namespace:System;assembly=netstandard">
    
    <Label Text="{x:Static sys:String.Empty}" />
</ContentPage>
```

### 15.4 x:Reference

Use this to reference another named object in the same XAML tree.

In the example below, the Label's Text property is bound to the Text property of the Entry control.

```xml
<ContentPage x:Class="MyApp.MainPage">
    <VerticalStackLayout>
        <Entry x:Name="nameEntry" />
        <Label Text="{Binding Source={x:Reference nameEntry}, Path=Text}" />
    </VerticalStackLayout>
</ContentPage>
```

### 15.5 x:Bind

`x:Bind` is a modern typed binding approach. It is usually used with the page or view model and is strongly typed.

```xml
<ContentPage
    x:Class="MyApp.MainPage"
    x:DataType="MyApp.MainPageViewModel">
 
    <Label Text="{x:Bind ViewModelName}" />
</ContentPage>
```

`x:Bind` is common in modern MAUI and is usually faster and more compile-time-safe than classic `Binding`.

## 16. Data binding and x:DataType

Binding connects UI to data. In MAUI, binding is one of the most important XAML features.

### Classic binding

```xml
<Label Text="{Binding UserName}" />
```

### Compiled binding with x:DataType

```xml
<ContentPage
    xmlns:vm="clr-namespace:MyApp.ViewModels"
    x:DataType="vm:MainViewModel">

    
    <Label Text="{Binding UserName}" />
</ContentPage>
```

`x:DataType` gives XAML a stronger type and improves tooling and compile-time checking.

## 17. XAML source generation

Modern .NET MAUI supports XAML source generation. This means the XAML compiler can generate strongly typed members and reduce manual boilerplate.

What this helps with:

- strongly typed `x:Name` references
- compile-time validation
- cleaner code-behind and better tooling
- improved performance and reliability

Typical pattern:

```xml
<ContentPage
    x:Class="MyApp.MainPage">

    <VerticalStackLayout>
        <Entry x:Name="nameEntry" />
        <Button x:Name="saveButton" Text="Save" />
    </VerticalStackLayout>
</ContentPage>
```

The generated code behind can then reference `nameEntry` and `saveButton` in a strongly typed way.

## 18. Platform differences in XAML

MAUI is cross-platform. You often want different values on iOS, Android, and Windows. Use `OnPlatform` or platform-specific resource values.

Example:
```xml
<Label
    Text="Platform-specific margin"
    Margin="{OnPlatform iOS='12,20,12,20', Android='12,16,12,16', WinUI='16,24,16,24'}" />
```

Another common pattern:

```xml
<Label Text="Hello" FontSize="{OnPlatform iOS=20, Android=18, WinUI=24}" />
```

This helps your app look correct on each platform without writing different XAML files per platform.

## 19. Common beginner mistakes

### 19.1 Missing x:Class or matching partial class

```xml
<ContentPage
    xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
    x:Class="MyApp.MainPage">
    <Label Text="Hello" />
</ContentPage>
```

```csharp
namespace MyApp;

public partial class MainPage : ContentPage
{
    public MainPage() => InitializeComponent();
}
```

### 19.2 Wrong property type

```xml
<!-- Wrong: IsVisible expects a boolean, not a string -->
<Label IsVisible="yes" />
```

Correct:

```xml

<Label IsVisible="True" />
```

### 19.3 Using names without `x:Name`

```xml

<Entry Placeholder="Name" />
```

To reference this in C#, you need:

```xml
<Entry x:Name="nameEntry" Placeholder="Name" />
```

## 20. Quick reference

### XAML essentials

```xml
<ContentPage x:Class="MyApp.MainPage">

    <VerticalStackLayout Padding="20" Spacing="12">
        <Label Text="Hello, world!" />
        <Entry x:Name="nameEntry" Placeholder="Type here" />
        <Button Text="Submit" Clicked="Submit_Clicked" />
    </VerticalStackLayout>
</ContentPage>
```

### Most common XAML syntax patterns

- `xmlns=".../maui"` = use the MAUI global/default namespace
- `xmlns:x=".../xaml"` = XAML directives namespace
- `x:Class="Namespace.ClassName"` = connects XAML to code-behind
- `x:Name="controlName"` = creates a named object reference
- `Text="Hello"` = set a property
- `{Binding Name}` = bind to data
- `{StaticResource PrimaryColor}` = read a shared resource
- `{DynamicResource AccentColor}` = read a resource that can change
- `{x:Static sys:String.Empty}` = access static values

### Common beginner control examples

```xml

<StackLayout Padding="20" Spacing="12">
    <Label Text="Name" />
    <Entry Placeholder="Type your name" />
    <Button Text="Save" />
    <CheckBox IsChecked="True" />
    <Picker Title="Pick one" />
</StackLayout>
```

## 21. Best practices for beginners

- Keep the MAUI default namespace in place for most files.
- Use `x:` only for XAML directives.
- Use `x:Name` when you need to work with a control in C#.
- Use `Binding` for view-model data.
- Use `StaticResource` for shared values that do not need to change.
- Use `DynamicResource` when values may update after initialization.
- Use `OnPlatform` for platform-specific differences.
- Use strong typing and compile-time binding when possible (`x:DataType`, `x:Bind`).

## 22. One-sentence summary

MAUI XAML is a way to declare your app’s UI in XML, using the MAUI namespace as the default/global namespace and the `x:` namespace for XAML directives; C# then adds behavior and logic.

## 23. Quick cheat sheet table

### Naming

- `xmlns="http://schemas.microsoft.com/dotnet/2021/maui"` = global MAUI namespace
- `xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"` = XAML namespace
- `x:Class` = link to code-behind class
- `x:Name` = generate a named field
- `local:` or `vm:` = custom named namespace prefixes (not required with using Global XML Namespaces)

### Typical beginner markup

```xml
<ContentPage x:Class="MyApp.MainPage">

    <VerticalStackLayout>
        <Label Text="Hello" />
        <Button Text="Tap me" />
    </VerticalStackLayout>
</ContentPage>
```

### Typical code-behind

```csharp
public partial class MainPage : ContentPage
{
    public MainPage()
    {
        InitializeComponent();
    }
}
```

### Key patterns to remember

- global/default namespace -> unprefixed controls
- `x:` prefix -> XAML directives
- `x:Class` + code-behind -> UI + behavior
- `x:Name` -> easier C# interaction
- `Binding` -> connect UI to data
- `StaticResource` / `DynamicResource` -> shared values
- `OnPlatform` -> platform differences
- `x:DataType` / `x:Bind` -> modern compiled binding patterns

