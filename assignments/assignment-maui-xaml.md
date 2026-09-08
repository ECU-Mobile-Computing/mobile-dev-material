# Assignment: Build a Single-Page .NET 11 MAUI XAML App

This assignment is a guided lab for students who have never used XML, XAML, or C# before. Students will build one single-page MAUI app that demonstrates the most important beginner XAML concepts in a beginner-friendly sequence.

The goal is not to build a production app. The goal is to build a simple app that makes the core XAML ideas visible and understandable.

## Learning goals
By the end of this assignment, students should be able to:

- explain what XAML is and why it is used in .NET MAUI
- understand the purpose of XML namespaces and the `x:` prefix
- explain the relationship between `x:Class` and the code-behind class
- use `x:Name` to access UI elements from C#
- use layout containers and common controls
- understand properties, property elements, attached properties, and content properties
- populate a page with shared resources, styles, and markup extensions
- use `StaticResource`, `DynamicResource`, and `x:Static`
- use `OnPlatform` for small platform differences
- understand the difference between global/default XAML namespaces and custom CLR namespaces
- recognize the modern .NET 11 MAUI XAML behavior with implicit namespace injection

## What students must submit
Each student should submit:

- the final MAUI app project
- a working page that runs on the assigned target platform
- comments in the XAML and C# code that explain the steps
- a final checklist confirming all required features are present

## Prerequisites
Students should already have:

- the .NET 11 preview SDK installed
- the MAUI workload installed
- the MAUI CLI working
- a working emulator or simulator for their platform
- a text editor or IDE with MAUI/XAML support

## Platform differences: Windows vs macOS vs Linux
Use the platform notes below during setup and testing.

### Windows
- Best experience: Visual Studio 2026 or current VS with MAUI workload installed.
- Android emulator is usually the easiest target for student work.
- Windows can also run MAUI Windows apps directly.
- If the emulator does not start, verify Android SDK installation and device creation.

### macOS
- Xcode is required for iOS and Mac Catalyst work.
- Students should use the iOS simulator or Mac Catalyst where available.
- Apple developer tools must be installed and accepted before simulator use.

### Linux
- Linux is suitable for Android development in many MAUI setups, but it is not the full equivalent of a Windows or macOS MAUI workstation.
- Students should not assume they can build or package iOS apps on Linux.
- They should use an Android emulator or a supported connected environment.

Important note for this assignment:
- The XAML concepts are the same across platforms.
- The UI should look the same in concept, but actual fonts, spacing, and platform controls may differ slightly.
- The app should compile and run on the platform the student is using.

## Assignment structure
Students must complete the steps in order. Do not skip steps. Each step is designed to build on the previous step.

The assignment uses a single page. A single page is enough to demonstrate all the major XAML concepts without creating a full app architecture.

## Step 1: Create the MAUI project
Create a new project.

```bash
mkdir MauiXamlLab
cd MauiXamlLab
dotnet new maui -n MauiXamlLab
cd MauiXamlLab
```

If your shell is already in the project folder, the command may be simplified. Use `dotnet new maui` in that folder if needed.

### What you are doing
This creates the default MAUI app template.

This template includes:

- `App.xaml`
- `App.xaml.cs`
- `MainPage.xaml`
- `MainPage.xaml.cs`
- `MauiProgram.cs`

### Verification
Open the project in VS Code or Visual Studio and confirm the project builds without errors.

Run the app.

Expected result:
- The default MAUI application loads.
- You see a default MAUI page with a button and text.

### Instructor note
This assignment intentionally starts from the default MAUI app so students learn how a MAUI page is organized before they rewrite it.

---

## Step 2: Open and understand the default XAML page
Open `MainPage.xaml`.

You should see something like this:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage x:Class="MauiXamlLab.MainPage">
    <ScrollView>
        <VerticalStackLayout
            Padding="30,0"
            Spacing="25">
            <Image Source="dotnet_bot.png" HeightRequest="185" Aspect="AspectFit" />
            <Label
                Text="Hello, World!"
                FontSize="32"
                HorizontalOptions="Center" />
            <Button
                Text="Click Me"
                Clicked="OnCounterClicked"
                HorizontalOptions="Center" />
        </VerticalStackLayout>
    </ScrollView>
</ContentPage>
```

### What this means
- `xmlns="http://schemas.microsoft.com/dotnet/2021/maui"` is the default MAUI namespace.
- Standard MAUI controls such as `ContentPage`, `Label`, `Button`, `ScrollView`, and `VerticalStackLayout` live in this namespace.
- The default namespace is a global XAML namespace. Controls in that namespace do not need a prefix.
- `xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"` is the XAML namespace.
- The `x:` prefix is used for XAML directives such as `x:Class`, `x:Name`, `x:Static`, `x:DataType`, and others.

### Important teaching point
This is the most important beginner concept:

- standard controls use the default/global MAUI namespace
- `x:` is for XAML-specific features


### Student task
Add a short comment above the `ContentPage` root explaining that the default namespace gives access to MAUI controls without a prefix.

```xml
<!-- The default MAUI namespace gives us access to controls like Label, Button, and Grid without a prefix. -->
```

---

## Step 3: Replace the default hello app with a beginner-friendly page
Open `MainPage.xaml` and replace the default page content with a simple page that includes a title, a button, and a status label.

### Write this code
```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage
    xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
    x:Class="MauiXamlLab.MainPage">

    <!-- This page uses the default MAUI namespace. Standard MAUI controls are used without a prefix. -->
    <VerticalStackLayout Padding="24" Spacing="16">
        <Label
            x:Name="TitleLabel"
            Text="My First MAUI XAML Page"
            FontSize="32"
            FontAttributes="Bold"
            HorizontalOptions="Center" />

        <Button
            x:Name="TapButton"
            Text="Tap me"
            Clicked="TapButton_Clicked" />

        <Label
            x:Name="StatusLabel"
            Text="Waiting for a click..."
            TextColor="#512BD4" />
    </VerticalStackLayout>
</ContentPage>
```

### What students should notice
- `x:Class` tells the XAML file which C# class it belongs to.
- `x:Name` creates a generated field so you can access the control in code.
- `Clicked="TapButton_Clicked"` wires the button event to a method in the code-behind.

### Open `MainPage.xaml.cs`
Write this code:

```csharp
namespace MauiXamlLab;

public partial class MainPage : ContentPage
{
    public MainPage()
    {
        // InitializeComponent loads the XAML UI into the C# partial class.
        InitializeComponent();
    }

    private void TapButton_Clicked(object sender, EventArgs e)
    {
        // x:Name created a field named StatusLabel. We can change its text here.
        StatusLabel.Text = "Hello from C#!";
    }
}
```

### What this demonstrates
- the relationship between `x:Class` and the code-behind class
- how `InitializeComponent()` connects the XAML and the C# partial class
- how `x:Name` gives a control a variable-like name in code
- a simple button event handler

### Verification
Run the app.

Expected behavior:
- the page appears
- the button click changes the status text

Important note:
- If the project does not build, check the namespace in `x:Class` and ensure the class name matches the code-behind class.

---

## Step 4: Add layout containers and explain properties
The next step introduces layout containers.

Replace the page with a page using `Grid` and `Border`.

### Code to add
```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage x:Class="MauiXamlLab.MainPage">

    <ScrollView>
        <VerticalStackLayout Padding="24" Spacing="16">

            <!-- Border is a container that can hold content. -->
            <Border Padding="16" Stroke="#512BD4" StrokeThickness="2" StrokeShape="RoundRectangle 12">
                <VerticalStackLayout Spacing="8">
                    <Label Text="Layout demo" FontSize="24" FontAttributes="Bold" />
                    <Label Text="This is inside a Border." />
                </VerticalStackLayout>
            </Border>

            <!-- Grid is a two-dimensional layout container. -->
            <Grid ColumnDefinitions="*,Auto" RowDefinitions="Auto,Auto" ColumnSpacing="12" RowSpacing="8">
                <Label Grid.Row="0" Grid.Column="0" Text="Name" FontAttributes="Bold" />
                <Entry Grid.Row="0" Grid.Column="1" Placeholder="Type your name" WidthRequest="180" />

                <Label Grid.Row="1" Grid.Column="0" Text="Notes" FontAttributes="Bold" />
                <Editor Grid.Row="1" Grid.Column="1" HeightRequest="100" WidthRequest="180" />
            </Grid>
        </VerticalStackLayout>
    </ScrollView>
</ContentPage>
```

### Explanation
- `VerticalStackLayout` arranges children vertically.
- `Border` is a container and its content is its content property.
- `Grid` is a common layout container.
- `Grid.Row` and `Grid.Column` are attached properties.
- `Padding`, `StrokeThickness`, `WidthRequest`, `HeightRequest`, and `FontAttributes` are ordinary properties.

### Teaching concept: attached properties
`Grid.Row` and `Grid.Column` are not defined on `Label` itself in the `Label` class. Rather, they belong to the `Grid` class and are attached to children.

This is a big beginner concept. Students should say:
- the child control is attached to a grid column or row
- the property is not a normal property of the child itself

### Verification
- The app should show a bordered section and a grid with labeled inputs.
- If layout is not correct, inspect spacing, `Grid.ColumnDefinitions`, and `Grid.RowDefinitions`.

---

## Step 5: Add common MAUI controls
This step adds the rest of the beginner-friendly controls.

Add the following controls to the same page inside the `VerticalStackLayout`:

```xml
<Label Text="Choose a theme" FontAttributes="Bold" />
<Picker Title="Pick a theme">
    <Picker.ItemsSource>
        <x:Array Type="{x:Type x:String}">
            <x:String>Light</x:String>
            <x:String>Dark</x:String>
            <x:String>System</x:String>
        </x:Array>
    </Picker.ItemsSource>
</Picker>

<Label Text="Choose a date" FontAttributes="Bold" />
<DatePicker />

<Label Text="Choose a time" FontAttributes="Bold" />
<TimePicker />

<Label Text="Volume" FontAttributes="Bold" />
<Slider Minimum="0" Maximum="100" Value="50" />

<HorizontalStackLayout Spacing="12">
    <CheckBox />
    <Label Text="Send updates" VerticalOptions="Center" />
</HorizontalStackLayout>

<Switch IsToggled="True" />
```

### What is happening here
Students are seeing the most common XAML controls:

- `Label`
- `Entry`
- `Editor`
- `Picker`
- `DatePicker`
- `TimePicker`
- `Slider`
- `CheckBox`
- `Switch`

### Verification
The page should show a set of common input controls without errors.

---

## Step 6: Explain properties, property elements, and content properties
This step introduces the difference between property syntax and property-element syntax.

### Regular property syntax
```xml
<Label Text="Hello" FontSize="18" FontAttributes="Bold" />
```

This is the normal beginner style and is the easiest way to write a property.

### Property element syntax
```xml
<Label Text="Hello">
    <Label.FontAttributes>Bold</Label.FontAttributes>
</Label>
```

This is valid XAML. It is more verbose, but it is useful for teaching that a property can be written as a nested element.

### Content property example
```xml
<Border Padding="16">
    <VerticalStackLayout>
        <Label Text="This is content" />
    </VerticalStackLayout>
</Border>
```

This is the content property pattern:
- the content of the `Border` is its child element
- the child is what appears inside the container
- the parent does not need an explicit `Content=` attribute


