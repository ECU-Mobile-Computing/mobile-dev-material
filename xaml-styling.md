# XAML Styling Guide for Beginners

This guide is for students who are brand new to XAML, XML, and C#. It focuses on the most important styling ideas in .NET MAUI and keeps the explanations beginner-friendly.

In modern MAUI examples, XML namespaces are often implicit, so the examples below omit `xmlns` declarations to keep the code simpler and easier to read.

The main idea: XAML lets you describe your UI as objects and properties, and styling lets you define consistent appearance for those objects.

## 1. What XAML is

XAML stands for Extensible Application Markup Language.

Think of it like this:

- A `Button` is an object
- `Text`, `BackgroundColor`, and `FontSize` are properties
- XAML writes those object/property relationships in a declarative way

Example:

```xml
<ContentPage>
    <VerticalStackLayout>
        <Label Text="Hello, XAML!" />
        <Button Text="Tap me" />
    </VerticalStackLayout>
</ContentPage>
```

This is not C# code. It is markup that describes the UI.

## 2. XML basics you need for XAML

XAML is based on XML, so a few rules matter:

- Elements use angle brackets: `<Button />`
- Attributes set values: `Text="Save"`
- Child elements go inside parent elements
- Elements must be properly closed
- XAML uses namespaces to know which controls are available

Example:

```xml
<Label Text="My label" FontSize="18" TextColor="Black" />
```

A beginner-friendly mental model:

- `Label` = control object
- `Text` = property
- `"My label"` = value
- `FontSize="18"` = another property/value pair

## 3. How XAML and C# work together

XAML usually describes the UI, while C# handles logic.

A common pattern is:

```xml
<ContentPage x:Class="MyApp.MainPage">
    <Button x:Name="saveButton"
            Text="Save"
            Clicked="saveButton_Clicked" />
</ContentPage>
```

And in C#:

```csharp
public partial class MainPage : ContentPage
{
    public MainPage()
    {
        InitializeComponent();
    }

    private void saveButton_Clicked(object sender, EventArgs e)
    {
        // Handle click logic here
    }
}
```

Important beginner concepts:

- `x:Class` = connect the XAML page to its C# code-behind
- `x:Name` = give an element a name so C# can find it
- `Clicked` = an event handler attached in XAML

You do not need to master all of C# to start styling, but you should understand that XAML is the UI description and C# is the behavior layer.

## 4. Where styling can be applied

There are many places to set styling in XAML. Here is the complete beginner-level list:

- Inline property values on a single control
- A `Style` applied directly to one element
- Page-level resources
- App-level resources
- A separate `ResourceDictionary`
- Implicit styles
- Keyed styles
- Theme resources and light/dark colors
- Styles that inherit from other styles (`BasedOn`)
- `Triggers` and `VisualStateManager`
- Control templates / custom controls (more advanced)

For beginners, the most important and commonly used techniques are:

1. Inline styling
2. `Style` with `Setter`
3. Page/app resources
4. Implicit styles
5. Theme/color resources

Everything else is useful, but not the first thing most students need to learn.

## 5. The most common beginner styling patterns

### 5.1 Inline styling

This is the simplest approach. You set properties directly on the control.

```xml
<Button Text="Save"
        BackgroundColor="#512BD4"
        TextColor="White"
        CornerRadius="12"
        Padding="16,10" />
```

Good for:

- one-off controls
- quick prototypes
- experimenting

Not great for:

- many buttons that should look the same
- maintaining a consistent design

This is the easiest way to understand the concept of styling because the look is directly on the element.

### 5.2 Styles with `Setter`

A `Style` is a reusable set of property values.

```xml
<ContentPage.Resources>
    <Style x:Key="PrimaryButtonStyle" TargetType="Button">
        <Setter Property="BackgroundColor" Value="#512BD4" />
        <Setter Property="TextColor" Value="White" />
        <Setter Property="CornerRadius" Value="12" />
        <Setter Property="Padding" Value="16,10" />
        <Setter Property="FontAttributes" Value="Bold" />
    </Style>
</ContentPage.Resources>
```

Then apply it:

```xml
<Button Text="Save"
        Style="{StaticResource PrimaryButtonStyle}" />
```

This is one of the most important beginner techniques.

Key ideas:

- `x:Key` gives the style a name so you can reference it
- `TargetType` says which control this style belongs to
- `Setter` sets a property value
- `StaticResource` tells XAML to look up the named resource

This is the standard way to make a group of controls share the same look.

## 6. App-level and page-level resources

Resources are reusable values stored in a dictionary.

### Page-level resources

```xml
<ContentPage.Resources>
    <Color x:Key="PrimaryColor">#512BD4</Color>

    <Style x:Key="PrimaryButtonStyle" TargetType="Button">
        <Setter Property="BackgroundColor" Value="{StaticResource PrimaryColor}" />
        <Setter Property="TextColor" Value="White" />
        <Setter Property="CornerRadius" Value="12" />
    </Style>
</ContentPage.Resources>
```

This is useful when a style is only needed on one page.

### App-level resources

App-wide styling is usually placed in `App.xaml` or a shared resource dictionary.

```xml
<Application.Resources>
    <Color x:Key="PrimaryColor">#512BD4</Color>

    <Style x:Key="PrimaryButtonStyle" TargetType="Button">
        <Setter Property="BackgroundColor" Value="{StaticResource PrimaryColor}" />
        <Setter Property="TextColor" Value="White" />
    </Style>
</Application.Resources>
```

This is useful for shared colors and common styles across the app.

### Resource dictionary files

As apps grow, you often move styles into a separate file such as:

- `Styles/DefaultStyles.xaml`
- `Resources/Colors.xaml`
- `Resources/Styles.xaml`

This is a good practice because it keeps the UI organized.

## 7. Implicit styles

An implicit style has no `x:Key`.

```xml
<ContentPage.Resources>
    <Style TargetType="Button">
        <Setter Property="BackgroundColor" Value="#512BD4" />
        <Setter Property="TextColor" Value="White" />
        <Setter Property="CornerRadius" Value="12" />
    </Style>
</ContentPage.Resources>
```

Now every `Button` in that page gets the same look automatically.

Use implicit styles when:

- you want all buttons on a page to look the same
- you want a consistent default look
- you do not need to apply it manually

Use keyed styles when:

- only a few elements need the design
- you want to choose which ones use the style

## 8. BasedOn: style inheritance

A style can inherit from another style.

```xml
<ContentPage.Resources>
    <Style x:Key="BaseButtonStyle" TargetType="Button">
        <Setter Property="CornerRadius" Value="12" />
        <Setter Property="Padding" Value="16,10" />
    </Style>

    <Style x:Key="PrimaryButtonStyle"
           TargetType="Button"
           BasedOn="{StaticResource BaseButtonStyle}">
        <Setter Property="BackgroundColor" Value="#512BD4" />
        <Setter Property="TextColor" Value="White" />
    </Style>
</ContentPage.Resources>
```

This is useful when you have several button styles but want them to share common values.

## 9. Colors, brushes, and theme resources

A strong beginner habit is to store colors in one place instead of repeating hex values everywhere.

```xml
<Color x:Key="PrimaryColor">#512BD4</Color>
<Color x:Key="PageBackgroundColor">#F5F5F5</Color>
<Color x:Key="TextPrimaryColor">#1F1F1F</Color>
```

Then use them like this:

```xml
<Button BackgroundColor="{StaticResource PrimaryColor}"
        TextColor="{StaticResource TextPrimaryColor}" />
```

For the app to support light and dark mode, use theme-aware values:

```xml
<Style TargetType="ContentPage">
    <Setter Property="BackgroundColor"
            Value="{AppThemeBinding Light=White, Dark=#121212}" />
</Style>
```

This is one of the best beginner habits for making apps feel polished.

## 10. Common controls and how beginners style them

### Button

```xml
<Style x:Key="PrimaryButtonStyle" TargetType="Button">
    <Setter Property="BackgroundColor" Value="#512BD4" />
    <Setter Property="TextColor" Value="White" />
    <Setter Property="FontAttributes" Value="Bold" />
    <Setter Property="CornerRadius" Value="12" />
    <Setter Property="Padding" Value="16,10" />
</Style>
```

### Label

```xml
<Style x:Key="TitleLabelStyle" TargetType="Label">
    <Setter Property="FontSize" Value="22" />
    <Setter Property="FontAttributes" Value="Bold" />
    <Setter Property="TextColor" Value="#1F1F1F" />
</Style>
```

### Entry

```xml
<Style x:Key="FieldStyle" TargetType="Entry">
    <Setter Property="HeightRequest" Value="50" />
    <Setter Property="Padding" Value="12,0" />
    <Setter Property="BackgroundColor" Value="White" />
    <Setter Property="TextColor" Value="#1F1F1F" />
</Style>
```

### Layout containers

```xml
<Style x:Key="CardStyle" TargetType="Border">
    <Setter Property="Stroke" Value="#D9D9D9" />
    <Setter Property="StrokeThickness" Value="1" />
    <Setter Property="Padding" Value="12" />
    <Setter Property="StrokeShape" Value="RoundRectangle 16" />
</Style>
```

Then:

```xml
<Border Style="{StaticResource CardStyle}">
    <VerticalStackLayout Spacing="8">
        <Label Text="Card title" />
        <Label Text="Some content here" />
    </VerticalStackLayout>
</Border>
```

This is a very common beginner pattern: create a small reusable style, then apply it to several controls.

## 10.5 Common properties to know

When students start learning XAML, it helps to know that some properties are shared across many controls and some are specific to a control category. This is not a complete list of every property in MAUI, but it is the list that shows up most often in beginner styling work.

### Shared properties for most controls

Most MAUI controls inherit from `VisualElement` or `View`, so the following properties are common across many controls:

- `BackgroundColor` or `Background`
- `TextColor` (for text-based controls)
- `Padding`
- `Margin`
- `WidthRequest` and `HeightRequest`
- `MinimumWidthRequest`, `MinimumHeightRequest`
- `MaximumWidthRequest`, `MaximumHeightRequest`
- `HorizontalOptions`, `VerticalOptions`
- `IsVisible`
- `IsEnabled`
- `Opacity`
- `Rotation`, `RotationX`, `RotationY`
- `Scale`, `ScaleX`, `ScaleY`
- `TranslationX`, `TranslationY`
- `AnchorX`, `AnchorY`
- `Clip`
- `Shadow`
- `CornerRadius` (on controls that support rounded corners)

These are the “big picture” properties students see first when styling a screen.

### Properties common to layout containers

Layout containers are used to arrange child controls. Common layout properties include:

- `Padding`
- `Margin`
- `Spacing` (for stack layouts)
- `BackgroundColor`
- `Orientation` (HorizontalStackLayout / VerticalStackLayout)
- `ColumnDefinitions` and `RowDefinitions` (Grid)
- `ColumnSpacing` and `RowSpacing` (Grid)
- `Children` (not styled directly, but used to add content)

Examples:

```xml
<VerticalStackLayout Padding="20" Spacing="12" BackgroundColor="LightGray">
    <Label Text="Hello" />
    <Button Text="Click me" />
</VerticalStackLayout>
```

This is one of the most common beginner patterns: put spacing and padding on a layout container so the child controls line up consistently.

### Properties common to text-based controls

Controls like `Label`, `Button`, `Entry`, `Editor`, and `SearchBar` often share text-related properties:

- `Text`
- `TextColor`
- `FontSize`
- `FontFamily`
- `FontAttributes`
- `TextTransform`
- `CharacterSpacing`
- `HorizontalTextAlignment`, `VerticalTextAlignment`
- `LineBreakMode` (common on `Label`)

Example:

```xml
<Label Text="Welcome"
       TextColor="#1F1F1F"
       FontSize="22"
       FontAttributes="Bold" />
```

### Properties common to button-like controls

Controls such as `Button`, `ImageButton`, and similar clickable controls often use:

- `Text`
- `TextColor`
- `BackgroundColor`
- `BorderColor`
- `BorderWidth`
- `CornerRadius`
- `Padding`
- `FontAttributes`
- `Command`
- `ImageSource` (for image buttons)

Example:

```xml
<Button Text="Save"
        BackgroundColor="#512BD4"
        TextColor="White"
        CornerRadius="12"
        Padding="16,10" />
```

### Properties common to input controls

Input controls such as `Entry`, `Editor`, and `SearchBar` often use:

- `Text`
- `Placeholder`
- `PlaceholderColor`
- `TextColor`
- `BackgroundColor`
- `Keyboard`
- `ReturnType`
- `IsPassword`
- `MaxLength`
- `ClearButtonVisibility` (some controls)
- `HeightRequest`

Example:

```xml
<Entry Placeholder="Email"
       PlaceholderColor="#777777"
       BackgroundColor="White"
       HeightRequest="50" />
```

### Properties unique to a few controls

These are common enough to memorize because they appear often in beginner MAUI projects:

- `Border` / `Frame`: `Stroke`, `StrokeThickness`, `StrokeShape`, `Shadow`, `Padding`
- `Grid`: `ColumnDefinitions`, `RowDefinitions`, `ColumnSpacing`, `RowSpacing`
- `StackLayout` / `VerticalStackLayout` / `HorizontalStackLayout`: `Spacing`, `Orientation`
- `Image`: `Source`, `Aspect`, `IsAnimationPlaying`
- `WebView`: `Source`
- `CollectionView`: `ItemsSource`, `ItemTemplate`, `SelectionMode`
- `CheckBox`: `IsChecked`, `Color`
- `Switch`: `IsToggled`, `OnColor`, `ThumbColor`
- `Slider`: `Minimum`, `Maximum`, `Value`

These are the “specialty” properties that belong to specific control categories, not all controls.

### Quick beginner rule of thumb

When you see a control in XAML, ask three questions:

1. Is this a text control, a layout container, or a button/input control?
2. Does it use a shared property like `BackgroundColor`, `Padding`, or `TextColor`?
3. Does it have a specialty property such as `Spacing`, `Stroke`, or `Placeholder`?

Once you learn those categories, styling becomes much easier to reason about.

## 11. Other styling-related features you may see

These are useful, but they are not the first things beginners need to memorize.

### `Triggers`

A trigger changes a property when a condition becomes true.

```xml
<Style TargetType="Button">
    <Setter Property="BackgroundColor" Value="#512BD4" />
    <Style.Triggers>
        <Trigger TargetType="Button"
                 Property="IsPressed"
                 Value="True">
            <Setter Property="BackgroundColor" Value="#3F2B8A" />
        </Trigger>
    </Style.Triggers>
</Style>
```

This is great for hover, focus, pressed, or selected states.

### `VisualStateManager`

Used for state-based UI changes such as normal, focused, disabled, selected.

This is more advanced and often used when building more polished control behavior.

### Control templates

Templates let you customize the visual structure of a control. This is powerful, but not usually where beginners start.

## 12. Best practices for beginners

- Prefer styles over repeating the same inline properties
- Start with a small set of app colors and spacing values
- Use page resources for page-specific appearance and app resources for cross-app styling
- Use `Style` to keep UI consistent
- Use `BasedOn` to reduce duplication
- Use implicit styles for a default control look
- Use keyed styles when you need to choose specific controls
- Add light/dark theme support early if possible
- Keep naming clear and consistent

## 13. Common beginner mistakes

- Forgetting to set `TargetType` in a style
- Using a `Style` with a `TargetType` that does not match the control
- Forgetting `x:Key` when trying to reference a named style
- Using inline styling everywhere instead of creating reusable styles
- Hard-coding colors in many places instead of using resources
- Expecting a style to apply when the control type does not match exactly

## 14. Quick beginner cheat sheet

Use this mental model:

- Inline styling = quick fix for one control
- Keyed style = reusable style you apply manually
- Implicit style = default look for all controls of a type
- App/page resources = reusable values and styles
- `BasedOn` = builds on an existing style
- Themes = light/dark-aware values

## 15. Simple example: full page

```xml
<ContentPage x:Class="MyApp.MainPage">

    <ContentPage.Resources>
        <Color x:Key="PrimaryColor">#512BD4</Color>
        <Color x:Key="PageBackgroundColor">#F5F5F5</Color>

        <Style TargetType="Button">
            <Setter Property="BackgroundColor" Value="{StaticResource PrimaryColor}" />
            <Setter Property="TextColor" Value="White" />
            <Setter Property="CornerRadius" Value="12" />
            <Setter Property="Padding" Value="16,10" />
        </Style>

        <Style x:Key="TitleLabelStyle" TargetType="Label">
            <Setter Property="FontSize" Value="24" />
            <Setter Property="FontAttributes" Value="Bold" />
            <Setter Property="TextColor" Value="#1F1F1F" />
        </Style>
    </ContentPage.Resources>

    <ScrollView BackgroundColor="{StaticResource PageBackgroundColor}">
        <VerticalStackLayout Padding="20" Spacing="12">
            <Label Text="Welcome" Style="{StaticResource TitleLabelStyle}" />
            <Entry Placeholder="Email" />
            <Button Text="Sign in" />
            <Button Text="Create account" />
        </VerticalStackLayout>
    </ScrollView>
</ContentPage>
```

This is a very common beginner layout: a shared page style, a few named styles, and controls that look consistent.

## 16. Final takeaway

The biggest beginner lesson is this:

- XAML describes the UI
- `Style` is how you make the UI consistent
- resources keep values reusable
- the most common and useful styling approaches are inline properties, page/app resources, keyed styles, and implicit styles

Once students understand those patterns, they are ready to build clear, consistent, and professional-looking MAUI apps.

If you want to go deeper later, the next topics are:

- `Triggers`
- `VisualStateManager`
- `DynamicResource`
- custom styles and themes
- control templates

But for a first pass, the techniques above are the ones you will use most often.
