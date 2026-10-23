# MAUI DevFlow Setup Guide

If MAUI DevFlow is not working, follow the steps below carefully. Complete each step and verify there are no errors before moving on.

---

## 1. Install the Latest MAUI VS Code Extension

In Visual Studio Code:

1. Open **Extensions**.
2. Search for:


```
ms-dotnettools.dotnet-maui
```

3. Install the extension if it is not already installed.
4. Enable **Use Prerelease Version**.
5. Verify the installed version is:


```
11.0.114
```

*(Current version as of October 22, 2026.)*

---

## 2. Install or Update the MAUI CLI

Open a terminal and run:

```bash
dotnet tool update -g Microsoft.Maui.Cli --prerelease
```

Verify the installation:

```bash
maui --version
```

3. Add the DevFlow Agent Package

From your MAUI project folder:
```bash
dotnet add package Microsoft.Maui.DevFlow.Agent --prerelease
```

4. Add the DevFlow Namespace

In MauiProgram.cs, add the following using statement wrapped in a DEBUG conditional:
```csharp
#if DEBUG
using Microsoft.Maui.DevFlow.Agent;
#endif
```
5. Register the DevFlow Agent

In MauiProgram.cs, add the DevFlow agent before the call to builder.Build():
```csharp
#if DEBUG
builder.AddMauiDevFlowAgent();
#endif
```

Example:
```csharp
var builder = MauiApp.CreateBuilder();

// Additional configuration...

#if DEBUG
builder.AddMauiDevFlowAgent();
#endif

return builder.Build();
```
6. Start the DevFlow Broker

From the MAUI project folder, run:
```bash
maui devflow broker start
```

Leave this terminal running.

7. Start the DevFlow MCP Server

Open a second terminal in the MAUI project folder.

Run the MCP server, selecting the platform you are targeting:

Android
```bash
maui devflow mcp --platform android
```
iOS
```bash
maui devflow mcp --platform ios
```
Windows
```bash
maui devflow mcp --platform windows
```
Mac Catalyst
```bash
maui devflow mcp --platform maccatalyst
```

Leave this terminal running.

8. Wait for the Agent Connection

Open a third terminal in the MAUI project folder:
```bash
maui devflow agent wait
```

This command waits for the mobile application to connect to DevFlow.

Leave this terminal running.

9. Launch the Application in DEBUG Mode

Start your MAUI application using the DEBUG configuration:
```bash
dotnet run -f <TARGET_FRAMEWORK> -c DEBUG
```

Examples:

Android
```bash
dotnet run -f net9.0-android -c DEBUG
```
Windows
```bash
dotnet run -f net9.0-windows10.0.19041.0 -c DEBUG
```

Important: DevFlow will not connect when running a Release build.

10. Verify the Agent Connects

Return to the terminal running:
```bash
maui devflow agent wait
```

You should see a message indicating that the agent has connected successfully.

If the agent does not connect:

- Confirm the application was started with -c DEBUG
- Confirm builder.AddMauiDevFlowAgent() was added to MauiProgram.cs
- Confirm the Broker is running
- Confirm the MCP Server is running
- Restart all terminals and try again

11. Open the DevFlow Inspector

Once the agent is connected, open the following URL in your browser:
```
http://localhost:19223/inspector
```

The DevFlow Inspector allows you to view and interact with the connected application.

Quick Verification Checklist

- MAUI VS Code extension installed (ms-dotnettools.dotnet-maui)
- Extension version is 11.0.114
- Using the latest prerelease extension
- MAUI CLI updated:
```bash
dotnet tool update -g Microsoft.Maui.Cli --prerelease
```

 - DevFlow package installed:
```bash
dotnet add package Microsoft.Maui.DevFlow.Agent --prerelease
```

 - Namespace added:
```csharp
#if DEBUG
using Microsoft.Maui.DevFlow.Agent;
#endif
```

 - Agent registered:
```csharp
#if DEBUG
builder.AddMauiDevFlowAgent();
#endif
```

 - Broker started:
```bash
maui devflow broker start
```

- MCP Server started- 
- Application launched in DEBUG mode- 
- Agent successfully connected- 
- Inspector accessible at:
```
http://localhost:19223/inspector
```
- Most Common Issue

> Note: The most common reason DevFlow does not connect is that the application was launched in Release mode instead of Debug mode.

Always start the application using:
```bash
dotnet run -c DEBUG
```

or a Visual Studio / VS Code launch profile configured for Debug.