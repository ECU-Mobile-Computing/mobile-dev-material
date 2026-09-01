# Assignment: Build a .NET 11 MAUI App with DevFlow

## Overview

This assignment introduces students to MAUI DevFlow, a testing, automation, and debugging toolkit for .NET MAUI apps. DevFlow helps developers inspect a running app, understand the visual tree, diagnose layout problems, automate UI interactions, capture screenshots, and work with the app through a local broker and browser-based inspector.

The goal of the assignment is not to build a large app. The goal is to teach students how to create a modern .NET 11 MAUI app, enable the DevFlow agent, and use a consistent set of DevFlow tools to inspect and manipulate the running UI.

This lab is intentionally focused on pure .NET MAUI. We are not using Blazor Hybrid or WebView-based app scenarios.

## Reference documentation

This assignment is based on the MAUI DevFlow references below, with a pure MAUI focus and no Blazor-specific work:

- https://github.com/dotnet/maui-labs/tree/main/src/DevFlow
- https://github.com/dotnet/maui-labs/blob/main/docs/DevFlow/inspector.md

## Learning goals

By the end of this assignment, students should be able to:

- explain what MAUI DevFlow is and why it matters
- describe the DevFlow architecture and how its pieces work together
- enable .NET 11 preview features and XAML source generation in a MAUI project
- add the DevFlow agent package to a MAUI app
- run the DevFlow CLI against a live app
- inspect the visual tree and diagnose layout issues
- query, tap, fill, and mutate elements at runtime
- capture screenshots and use the browser-based inspector
- connect DevFlow to the broader AI/MCP workflow with `maui devflow init --target github`

## Why DevFlow matters

When students build MAUI apps, they often run into problems like:

- a button is clipped or hidden
- text is truncated or overlapped
- a page layout looks different than expected
- an element is not where the developer thinks it is
- a control was added, but the visual tree does not match the code

DevFlow solves these problems by letting developers inspect the app while it is running. Instead of guessing, they can query the tree, inspect properties, simulate user interaction, and see the UI as the app sees it.

## DevFlow components

DevFlow is made up of several connected parts:

- Agent: the in-app runtime component embedded into a MAUI app. It exposes the visual tree, element properties, diagnostics, screenshots, and interaction endpoints.
- Broker: the local coordination service that connects the app to inspector and CLI clients. It hosts the DevFlow browser-based UI and routes live requests to the selected agent.
- CLI: the `maui devflow` command surface used for local inspection, automation, and tooling.
- Browser-based inspector: a local web UI served at `http://localhost:19223/inspector/` that shows the app screenshot and visual tree overlay.
- MCP: the Model Context Protocol server that exposes DevFlow tools to AI tooling agents and other automation clients.

Together, these components let students inspect and control a running app from the terminal, a web page, or an AI-enabled workflow.

## Common tasks DevFlow solves

Students will use DevFlow to complete tasks such as:

- inspect the live visual tree with `maui devflow ui tree`
- detect clipping, overflow, text truncation, and overlap with `maui devflow ui diagnostics`
- query elements by type or selector to find the right control
- tap a button or element at runtime without manually interacting with the app
- fill text into an `Entry` or editor field
- read and change properties such as `Text`, `TextColor`, and `IsVisible`
- capture screenshots for troubleshooting and documentation
- open the web inspector to visually inspect the running UI
- automate app interaction for learning or testing scenarios
- connect DevFlow with GitHub Copilot or other AI workflow tooling through MCP

## Prerequisites

Students should already have:

- .NET 11 preview SDK installed
- MAUI workload installed
- MAUI CLI available
- a working Android emulator, Windows app target, or simulator
- an IDE such as VS Code or Visual Studio with MAUI support
- a basic understanding of C# and XAML

## Project setup

Using the existing MauiXamlLab from assignment-maui-xaml, start from the root of the maui project.  Enable the preview features required for the newest .NET 11/XAML experience.


Open the project file and update the `PropertyGroup` section to include the following additional elements:
- EnablePreviewFeatures
- MauiXamlInflator

```xml
<PropertyGroup>
    <TargetFrameworks>net11.0-android;net11.0-ios;net11.0-maccatalyst</TargetFrameworks>
    <OutputType>Exe</OutputType>
    <RootNamespace>MauiDevFlowLab</RootNamespace>
    <UseMaui>true</UseMaui>
    <SingleProject>true</SingleProject>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <EnablePreviewFeatures>true</EnablePreviewFeatures>
    <MauiXamlInflator>SourceGen</MauiXamlInflator>
</PropertyGroup>
```

Important note:

- `EnablePreviewFeatures` turns on the newest preview behavior for .NET 11 and MAUI features.
- `MauiXamlInflator` tells the MAUI XAML pipeline to use the source-generated inflator model.
- The exact target framework can vary by platform, but the assignment is designed for the .NET 11 preview environment.

## Install/update the MAUI CLI

Update to the latest MAUI CLI prerelease.

```bash
dotnet tool update microsoft.maui.cli -g --prerelease
```

This ensures students are using the newest CLI features for DevFlow and related tooling.

## Install the DevFlow agent package

Add the DevFlow agent package to the project:

```bash
dotnet add package Microsoft.Maui.DevFlow.Agent --prerelease
```

This package injects the in-app DevFlow agent and exposes the runtime endpoints used by the CLI, inspector, and MCP tooling.

## Register the DevFlow agent in `MauiProgram.cs`

Open `MauiProgram.cs` and register the agent before `builder.Build()`.

```csharp
using Microsoft.Maui.DevFlow.Agent;

public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        builder.UseMauiApp<App>();

        if (app.Environment.IsDevelopment())
        {
            builder.AddMauiDevFlowAgent(options =>
            {
                options.EnableLayoutDiagnostics = true;
            });
        }

        return builder.Build();
    }
}
```

This is the key runtime registration that enables DevFlow functionality inside the MAUI app. You can also keep this in a debug-only block for release builds if you want to disable it outside development.

## Initialize DevFlow skills

Once the app is ready, initialize the DevFlow skills for the target environment.

```bash
maui devflow init --target github
```

This initializes the tooling needed for MCP/AI workflows and adds the relevant DevFlow skill files for the selected host environment.

## Sample app for this assignment

Create a simple page that has a button and a status label. Give the button a fixed `AutomationId` so it is easy to target with DevFlow commands.

### `MainPage.xaml`

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage
    xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
    xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
    x:Class="MauiDevFlowLab.MainPage">

    <VerticalStackLayout Padding="24" Spacing="20">
        <Label
            Text="DevFlow Lab"
            FontSize="32"
            FontAttributes="Bold"
            HorizontalOptions="Center" />

        <Border
            Stroke="#512BD4"
            StrokeThickness="2"
            Padding="18"
            StrokeShape="RoundRectangle 12">
            <VerticalStackLayout Spacing="12">
                <Label
                    x:Name="CounterLabel"
                    Text="Count: 0"
                    FontSize="24" />

                <Button
                    x:Name="CounterBtn"
                    AutomationId="CounterBtn"
                    Text="Increment"
                    Clicked="OnCounterClicked" />

                <Label
                    x:Name="StatusLabel"
                    Text="Ready. Use DevFlow to inspect this page."
                    TextColor="DarkSlateGray" />
            </VerticalStackLayout>
        </Border>
    </VerticalStackLayout>
</ContentPage>
```

### `MainPage.xaml.cs`

```csharp
namespace MauiDevFlowLab;

public partial class MainPage : ContentPage
{
    private int _count;

    public MainPage()
    {
        InitializeComponent();
    }

    private void OnCounterClicked(object sender, EventArgs e)
    {
        _count++;
        CounterLabel.Text = $"Count: {_count}";
        StatusLabel.Text = $"Button tapped {_count} time(s).";
    }
}
```

This is intentionally simple. The value is that students can later use DevFlow to locate the button, inspect the label, and verify the state updates in real time.

## Run the app

Run the app in a supported target platform. For a Windows-based lab, this is a common command:

```bash
dotnet watch run -f net11.0-windows10.0.19041.0
```

If students are targeting a different platform, they should use the matching MAUI target framework and emulator or simulator.

## DevFlow features

DevFlow includes a broad set of capabilities. In this assignment, students will focus on the core MAUI features below.

### Visual tree inspection

DevFlow can inspect the running UI hierarchy and list the elements that make up the page.

Example:

```bash
maui devflow ui tree
```

This helps students answer questions such as:

- What controls exist in this page?
- Which container holds the button?
- Is the layout nested the way I expect?

### Layout diagnostics

DevFlow can detect clipping, overflow, text truncation, overlap, and occlusion.

Example:

```bash
maui devflow ui diagnostics
```

Students can use this to find UI bugs before they become visible to users. In a classroom setting, this is one of the best ways to show the value of DevFlow in real-time troubleshooting.

### Querying controls

Students can locate UI elements by type or selector.

Examples:

```bash
maui devflow ui query --type Label
maui devflow ui query --type Label --selector 'Border  Label'
```

These commands are useful when a page contains many controls and the developer needs a targeted way to find them.

### Tapping and interaction

DevFlow can trigger taps and other UI actions without using a mouse or a real user interaction.

Examples:

```bash
maui devflow ui tap "CounterBtn"
maui devflow ui tap --automationId "CounterBtn"
```

The `AutomationId` approach is more reliable and easier to teach because it is explicit and stable.

### Filling and clearing input

For text fields and editor controls, DevFlow can fill or clear values.

Example:

```bash
maui devflow ui fill <elementId> "Hello DevFlow"
maui devflow ui clear <elementId>
```

Not all command forms are identical in every version, so students should run `maui devflow ui --help` and confirm the exact syntax supported by their local CLI.

### Screenshot capture

DevFlow can capture an image of the running app.

Example:

```bash
maui devflow ui screenshot --output screenshot.png
```

This is useful for bug reports, UI review, and assignment verification.

### Runtime property inspection and live editing

DevFlow can read and set runtime properties, making it useful for quick experiments.

Examples:

```bash
maui devflow ui property <elementId> TextColor
maui devflow ui set-property <elementId> TextColor Blue
```

Students can verify that a `Label` or `Button` property changes immediately, which teaches how XAML and runtime values are related.

### Browser inspector

The browser-based DevFlow inspector presents the running app as a screenshot plus a visual-tree overlay. It is served at:

```text
http://localhost:19223/inspector/
```

Use this to:

- inspect the live visual tree
- click an element to inspect its metadata
- look at layout issues visually
- perform live edits and gesture testing
- understand how the visual tree maps to the UI

### MCP and AI integration

DevFlow also includes an MCP server for AI-driven workflows.

Example:

```bash
maui devflow mcp
```

This exposes DevFlow to AI tools and other automation clients that understand MCP.

### Broker and session management

The broker allows DevFlow clients to connect to a running app and coordinate access to the same app session.

Example:

```bash
maui devflow broker start
```

This service is the foundation for the browser inspector and other connected tooling.

## DevFlow command reference

These are the most important commands students should learn for this assignment.

### Core status and connection

- `maui devflow status` — checks whether a DevFlow agent is connected and available.
- `maui devflow broker start` — starts the DevFlow broker.
- `maui devflow mcp` — starts the MCP server for AI/automation clients.

Example:

```bash
maui devflow status
maui devflow broker start
maui devflow mcp
```

### UI inspection commands

- `maui devflow ui tree` — dumps the running visual tree.
- `maui devflow ui diagnostics` — detects clipping, overflow, text truncation, overlap, and occlusion.
- `maui devflow ui query --type Label` — lists label elements.
- `maui devflow ui query --type Label --selector 'Border  Label'` — finds a nested label using a selector.
- `maui devflow ui screenshot --output screenshot.png` — captures a screenshot.
- `maui devflow ui property <elementId> <propertyName>` — reads a runtime property.
- `maui devflow ui set-property <elementId> <propertyName> <value>` — mutates a live property.
- `maui devflow ui element <elementId>` — gets full metadata for a specific element.
- `maui devflow ui hit-test <x> <y>` — finds the element under a given point.

Examples:

```bash
maui devflow ui tree
maui devflow ui diagnostics
maui devflow ui query --type Label
maui devflow ui screenshot --output screenshot.png
maui devflow ui set-property 5b1c8f9edba5 TextColor Blue
```

### Interaction commands

- `maui devflow ui tap <elementId>` — taps an element.
- `maui devflow ui tap --automationId "CounterBtn"` — taps by automation ID.
- `maui devflow ui fill <elementId> <text>` — writes text into input controls.
- `maui devflow ui clear <elementId>` — clears a text field.
- `maui devflow ui focus <elementId>` — sets focus to an element.
- `maui devflow ui navigate <route>` — navigates to a Shell route.
- `maui devflow ui scroll` — scrolls content by delta or target location.
- `maui devflow ui gesture <gestureType>` — performs gestures such as tap, pan, swipe, pinch, and rotate.
- `maui devflow ui resize <width> <height>` — resizes the app window.
- `maui devflow ui alert` — detects and dismisses dialog alerts.
- `maui devflow ui assert <propertyName> <expectedValue>` — asserts property values.
- `maui devflow ui permission` — manages permissions on supported platforms.

Examples:

```bash
maui devflow ui tap "CounterBtn"
maui devflow ui tap --automationId "CounterBtn"
maui devflow ui fill 58a4f71d2ff3 "Hello"
maui devflow ui focus 58a4f71d2ff3
maui devflow ui gesture swipe --direction down
```

Note: command syntax can vary slightly by version. Students should use `--help` for the exact current parameter set.

## Browser inspector workflow

After the app is running and the broker is started, open the browser inspector:

```text
http://localhost:19223/inspector/
```

Students should use the inspector to:

1. confirm the app is connected to the broker
2. inspect the visual tree
3. select a control and read its properties
4. change a property live to test expected behavior
5. capture a screenshot for their notes
6. verify layout issues or visual structure

The inspector gives students a visual map between the app and the underlying MAUI tree, which makes debugging far easier than relying on console output alone.

## Assignment instructions

### Step 1: Create the project

Create a new .NET 11 MAUI project and enable the required configuration.

### Step 2: Enable DevFlow

- update the MAUI CLI
- add the DevFlow agent NuGet package
- register the agent in `MauiProgram.cs`
- initialize DevFlow skills with `maui devflow init --target github`

### Step 3: Build the demo UI

Create a simple page that includes:

- a page title
- a `Border` container
- a `Label` that displays a counter
- a primary `Button` with `AutomationId="CounterBtn"`
- a status label

### Step 4: Run and inspect the app

Use the following commands:

```bash
maui devflow status
maui devflow ui tree
maui devflow ui diagnostics
maui devflow ui query --type Label
maui devflow ui screenshot --output screenshot.png
```

### Step 5: Trigger interactions with DevFlow

Use the following commands to interact with the app:

```bash
maui devflow ui tap "CounterBtn"
maui devflow ui tap --automationId "CounterBtn"
```

Then inspect the result by checking the label text changes or by capturing a second screenshot.

### Step 6: Change a property at runtime

Use the UI inspector or the CLI to locate an element and change a property live.

Example:

```bash
maui devflow ui set-property <elementId> TextColor Blue
```

Then explain what changed and why the property was valid at runtime.

### Step 7: Open the browser inspector

Start the broker and open the inspector in the browser:

```bash
maui devflow broker start
```

Then visit:

```text
http://localhost:19223/inspector/
```

Use the inspector to verify that the chosen element and property values match the app you are running.

## Suggested reflection questions

Students should answer these questions in their notebook or submission:

1. What is the difference between the CLI and the browser inspector?
2. Why is `AutomationId` useful when debugging a MAUI app with DevFlow?
3. Why is layout diagnostics important in MAUI UIs?
4. How does DevFlow help locate a UI bug without reading every XAML line manually?
5. What kinds of apps would benefit most from DevFlow in a real development workflow?

## Student deliverables

Each student should submit:

- the final MAUI project
- a working app that runs with DevFlow enabled
- at least one screenshot captured using DevFlow
- notes showing the commands used during inspection
- a short written explanation of how DevFlow helped find or fix a UI issue
- a final checklist confirming the project meets the requirements

## Final checklist

Students should verify all of the following before submission:

- [ ] .NET 11 preview features enabled in the project
- [ ] `MauiXamlInflator` set to `SourceGen`
- [ ] MAUI CLI updated to the latest prerelease
- [ ] `Microsoft.Maui.DevFlow.Agent` added to the project
- [ ] the DevFlow agent registered in `MauiProgram.cs`
- [ ] `maui devflow init --target github` completed
- [ ] the app runs successfully on the assigned target platform
- [ ] `maui devflow ui tree` works
- [ ] `maui devflow ui diagnostics` runs
- [ ] `maui devflow ui tap --automationId "CounterBtn"` works
- [ ] a screenshot was captured using DevFlow
- [ ] the browser inspector was opened at `http://localhost:19223/inspector/`
- [ ] a property was changed or inspected live at runtime

## Instructor notes

This assignment is intentionally focused on learning by inspection and experimentation. DevFlow is most powerful when students can connect a real app to a live inspection and control workflow. The goal is to make UI debugging feel concrete and immediate rather than abstract.

Teachers can extend the lab by:

- adding more controls and nested layouts
- creating a page with a clipped or misaligned layout and then using diagnostics to find the issue
- asking students to use `query` and `set-property` to identify a missing or misconfigured control
- having students compare a working and broken layout using DevFlow output

## Quick cheat sheet

```bash
# Update the MAUI CLI
 dotnet tool update microsoft.maui.cli -g --prerelease

# Add the DevFlow agent
 dotnet add package Microsoft.Maui.DevFlow.Agent --prerelease

# Initialize DevFlow skills
 maui devflow init --target github

# Run the app
 dotnet watch run -f net11.0-windows10.0.19041.0

# Inspect the UI
 maui devflow ui tree
 maui devflow ui diagnostics
 maui devflow ui query --type Label
 maui devflow ui query --type Label --selector 'Border  Label'

# Interact with the app
 maui devflow ui tap "CounterBtn"
maui devflow ui tap --automationId "CounterBtn"

# Capture a screenshot
 maui devflow ui screenshot --output screenshot.png

# Inspect and edit runtime properties
 maui devflow ui property <elementId> TextColor
 maui devflow ui set-property <elementId> TextColor Blue

# Start the broker and open the browser inspector
 maui devflow broker start
# open: http://localhost:19223/inspector/

# Start MCP tooling
 maui devflow mcp
```

## Final takeaway

DevFlow turns MAUI UI debugging into a live, practical workflow. Instead of editing XAML and hoping for the right result, students can inspect the tree, query runtime elements, diagnose layout problems, trigger interactions, and verify app state in real time. This makes it one of the most effective tools for modern MAUI learning and troubleshooting.
