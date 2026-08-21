# .NET MAUI Prerelease Setup Handout

This handout is designed for students learning cross-platform mobile development with .NET MAUI using the latest .NET 11 preview and prerelease MAUI tooling.

## 1. Why this setup?

The goal is to install and validate the newest preview SDK and tooling so you can:

- build .NET MAUI apps with the newest .NET 11 preview features
- use the prerelease MAUI CLI for environment validation
- target Android, Windows, iOS, and Mac Catalyst when supported by your machine
- configure VS Code for productive MAUI development

## 2. Install the .NET 11 preview SDK

Download the latest .NET 11 preview SDK:

- https://dotnet.microsoft.com/en-us/download/dotnet/11.0

Check your installation:

```bash
dotnet --version
dotnet --list-sdks
```

You should see a .NET 11 preview release.

## 3. Install the prerelease MAUI CLI

```bash
dotnet tool install -g microsoft.maui.cli --prerelease
```

Verify:

```bash
maui --version
```

Inspect your machine:

```bash
maui doctor
```

If the tool reports missing components:

```bash
maui doctor --fix
```

## 4. Install the MAUI workload

```bash
dotnet workload install maui --include-previews
```

Verify:

```bash
dotnet workload list
```

## 5. Install Visual Studio Code

- Download: https://code.visualstudio.com/download

Create a profile named `mobile-dev`:

```bash
code --profile mobile-dev
```

Enable prerelease package support in VS Code:

- Open Settings
- Search for `C# Dev Kit`
- Enable `NuGet: Allow Prerelease Versions`

Install these extensions:

```bash
code --profile mobile-dev --install-extension alexcvzz.vscode-sqlite
code --profile mobile-dev --install-extension humao.rest-client
code --profile mobile-dev --install-extension jaufrdevosse.litedb-vscode
code --profile mobile-dev --install-extension microsoft-aspire.aspire-vscode
code --profile mobile-dev --install-extension ms-azuretools.vscode-containers
code --profile mobile-dev --install-extension ms-azuretools.vscode-docker
code --profile mobile-dev --install-extension ms-dotnettools.csdevkit
code --profile mobile-dev --install-extension ms-dotnettools.csharp
code --profile mobile-dev --install-extension ms-dotnettools.dotnet-maui
code --profile mobile-dev --install-extension ms-dotnettools.vscode-dotnet-runtime
code --profile mobile-dev --install-extension ms-mssql.data-workspace-vscode
code --profile mobile-dev --install-extension ms-mssql.mssql
code --profile mobile-dev --install-extension ms-mssql.sql-database-projects-vscode
code --profile mobile-dev --install-extension ms-vscode-remote.remote-containers
code --profile mobile-dev --install-extension ms-vscode.remote-explorer
code --profile mobile-dev --install-extension ms-vscode.remote-server
```

Enable the MCP marketplace in VS Code and add:

- Microsoft Learn
- Microsoft NuGet

## 6. Platform-specific setup

### Windows

Download Visual Studio 2026 Community:

- https://visualstudio.microsoft.com/downloads

Install only these workloads:

- MAUI development
- Windows C++ development

Also install the Windows target framework support and runtime support needed by MAUI.

### macOS

Install or update Xcode and the Apple developer tools.

Accept the license:

```bash
sudo xcodebuild -license accept
```

If needed:

```bash
xcode-select --install
```

### Linux

Use a supported Linux distribution with the .NET 11 preview SDK and the MAUI workload installed.

Important: Linux can be used for MAUI Android development, but not as a full MAUI target platform for packaging or deployment.

## 7. Create an Android emulator

Use the MAUI CLI to create the newest supported Android emulator for your machine:

```bash
maui emulators --help
maui emulators list
maui emulators create android --name "Pixel_11_API_35"
```

Start it:

```bash
maui emulators start android --name "Pixel_11_API_35"
```

If the exact command names differ in your preview version, use `--help` to confirm the current syntax.

## 8. Create a macOS simulator

This is for macOS only.

```bash
maui sim --help
maui sim create ios
maui sim list
maui sim launch "iPhone 16 Pro"
```

Use `--help` if the command format differs in your preview build.

## 9. Create a MAUI project with .NET 11 preview

Create a folder:

```bash
mkdir FirstMauiApp
cd FirstMauiApp
```

Create a `global.json` file:

```json
{
  "sdk": {
    "version": "11.0.100-preview.7.26381.103",
    "allowPrerelease": true,
    "rollForward": "latestMinor"
  }
}
```

Create the app:

```bash
dotnet new maui -o FirstMauiApp
```

Or, if already inside the folder:

```bash
dotnet new maui
```

## 10. Configure target frameworks

Use the correct target frameworks and runtime identifiers for your platform.

### Windows

```xml
<PropertyGroup>
  <TargetFrameworks>net11.0-android;net11.0-windows10.0.19041.0</TargetFrameworks>
  <RuntimeIdentifiers>android-arm;android-arm64;win-x64;win-arm64</RuntimeIdentifiers>
</PropertyGroup>
```

### macOS

```xml
<PropertyGroup>
  <TargetFrameworks>net11.0-android;net11.0-ios;net11.0-maccatalyst</TargetFrameworks>
  <RuntimeIdentifiers>android-arm;android-arm64;ios-arm64;iossimulator-x64;iossimulator-arm64;maccatalyst-x64;maccatalyst-arm64</RuntimeIdentifiers>
</PropertyGroup>
```

### Linux

```xml
<PropertyGroup>
  <TargetFrameworks>net11.0-android</TargetFrameworks>
  <RuntimeIdentifiers>android-arm;android-arm64</RuntimeIdentifiers>
</PropertyGroup>
```

## 11. Build the project

```bash
cd FirstMauiApp
dotnet build
```

Build a specific target if needed:

```bash
dotnet build -f net11.0-android
dotnet build -f net11.0-windows10.0.19041.0
dotnet build -f net11.0-ios
dotnet build -f net11.0-maccatalyst
```

## 12. Run the app

### Android emulator

- create and start the emulator
- build the Android target
- deploy and run the app on the emulator

### iOS simulator (macOS only)

- start the simulator
- build the iOS target
- run the app on the simulator

### Physical device

For Android:

- enable Developer Options and USB debugging
- connect the device
- authorize it
- build and deploy the Android target

For iOS:

- connect the iPhone or iPad
- trust the device
- use valid Apple signing configuration
- deploy the iOS app from the Apple toolchain

### Local workstation

On Windows, run the Windows target locally. On macOS, run the Mac Catalyst target or iOS simulator when supported.

## 13. Validation checklist

Before you continue to real app development, confirm:

- `dotnet --version` shows .NET 11 preview
- `maui --version` works
- `maui doctor` reports no critical issues
- `dotnet workload list` includes MAUI preview workloads
- Android emulator starts successfully
- macOS simulator starts successfully on Apple hardware
- sample MAUI app builds successfully
- app runs on at least one supported target

## 14. Quick cheat sheet

```bash
dotnet --version
dotnet --list-sdks

dotnet tool install -g microsoft.maui.cli --prerelease
maui --version
maui doctor
maui doctor --fix

dotnet workload install maui --include-previews
dotnet workload list

code --profile mobile-dev

mkdir FirstMauiApp
cd FirstMauiApp

dotnet new maui

dotnet build
```

## 15. Final note

This setup is intentionally preview-focused. The exact command names and supported emulator images may change as the MAUI tooling evolves, so using `--help` and `maui doctor` is part of the normal workflow.
