# MAUI Prerelease Environment Setup Guide (.NET 11 + MAUI Preview)

This guide is intended for a Windows, macOS, or Linux workstation that will be used for .NET MAUI development with the latest .NET 11 preview SDK and the prerelease MAUI CLI.

The goal is to install the correct preview tooling, configure VS Code for MAUI work, create a .NET MAUI app, and validate that the environment can build and run on supported targets.

Important notes:

- .NET 11 is preview software and may change before GA.
- The MAUI CLI is a prerelease tool and usually requires the latest preview .NET SDK.
- Use the newest supported Android emulator or simulator for your platform.
- Linux can be used for MAUI development, but MAUI apps are only supported for Android on Linux. Windows and macOS support broader target sets.

## 1. Install the .NET 11 preview SDK

Download the latest .NET 11 preview SDK from Microsoft:

- https://dotnet.microsoft.com/en-us/download/dotnet/11.0

After installation, confirm the SDK is available:

```bash
dotnet --version
dotnet --list-sdks
```

You should see a .NET 11 preview release in the list.

If you want to pin the repo to a specific preview version, create a `global.json` file in the project folder:

```json
{
  "sdk": {
    "version": "11.0.100-preview.7.26381.103",
    "allowPrerelease": true,
    "rollForward": "latestMinor"
  }
}
```

This ensures the project resolves to a matching .NET 11 preview SDK when opened from that directory.

## 2. Install the MAUI CLI prerelease

Install the prerelease MAUI CLI globally:

```bash
dotnet tool install -g microsoft.maui.cli --prerelease
```

If you already installed it and need to update to the newest preview:

```bash
dotnet tool update -g microsoft.maui.cli --prerelease
```

Verify the installation:

```bash
maui --version
```

## 3. Use MAUI CLI to validate and repair the environment

Run the MAUI doctor command to inspect the machine for missing components:

```bash
maui doctor
```

If the tool reports missing workloads, workloads, emulator support, or platform dependencies, fix them with:

```bash
maui doctor --fix
```

This is typically the fastest way to identify the required pieces for Android, iOS, or Windows development.

## 4. Install the .NET MAUI workload

Install the MAUI workload using the preview channel:

```bash
dotnet workload install maui --include-previews
```

Confirm the installed workloads:

```bash
dotnet workload list
```

This should show MAUI-related workloads, including the preview workload set.

## 5. Install VS Code and create a MAUI profile

Download VS Code:

- https://code.visualstudio.com/download

Install the latest stable release, then use a dedicated profile for mobile work.

Open a terminal and create or select a profile named `mobile-dev`:

```bash
code --profile mobile-dev
```

If the profile does not exist yet, VS Code will create it when opened using that profile name.

Enable prerelease package support inside VS Code for C# tooling:

- Open Settings
- Search for `C# Dev Kit`
- Enable `NuGet: Allow Prerelease Versions`

Install the recommended extensions for this profile:

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

Then enable the MCP marketplace in VS Code and add the following MCP servers:

- Microsoft Learn
- Microsoft NuGet

Switch to the MAUI agent in your chat/tooling environment, then restart VS Code so the new profile and extensions are loaded.

## 6. Platform-specific installation steps

### Windows workstation installation

Download and install the latest Visual Studio 2026 Community edition:

- https://visualstudio.microsoft.com/downloads

During installation, select only these workloads:

- MAUI development
- Windows C++ development

Also make sure the Windows target platform and runtime support are installed, including the Windows target framework bits for MAUI. Without the Windows C++ tools and SDK support, the Windows MAUI app cannot build successfully.

After installation, validate the Windows development tools:

```bash
dotnet --list-sdks
dotnet workload list
```

### macOS workstation installation

Use the latest Xcode and Apple toolchain supported by the .NET 11 preview. Install or update Xcode from the App Store or Apple developer tools, then accept the license agreement:

```bash
sudo xcodebuild -license accept
```

If you are using Xcode command line tools:

```bash
xcode-select --install
```

Install the .NET 11 preview SDK, then install the MAUI workload as shown above.

### Linux workstation installation

Use the latest supported Linux distribution for MAUI preview development, then install the .NET 11 preview SDK and the MAUI workload.

Important: Linux is suitable for building and testing Android workloads, but MAUI app support on Linux is limited to Android. It is not a full replacement for the Windows or macOS app-building environment.

## 7. Create an Android emulator using the MAUI CLI

On Windows, macOS, and Linux, create the latest supported Android emulator for MAUI using the MAUI CLI.

The exact flags may vary by preview build, so always check the CLI help first if your version differs:

```bash
maui emulators --help
```

Typical flow:

```bash
maui emulators list
maui emulators create android --name "Pixel_11_API_35"
```

If your preview CLI uses a different syntax, use the command help to find the current parameters. The key objective is to create a supported Android emulator image that works with the MAUI preview workload.

After creation, start and validate it:

```bash
maui emulators start android --name "Pixel_11_API_35"
maui emulators list
```

You can also use Android Studio's Device Manager if the MAUI CLI does not expose the exact latest image in your version.

## 8. Create a macOS simulator using the MAUI CLI

On macOS only, create the newest simulator supported by MAUI preview:

```bash
maui sim --help
```

Then create and launch a simulator using the MAUI CLI or the Apple tools. A typical pattern is:

```bash
maui sim create ios
maui sim list
maui sim launch "iPhone 16 Pro"
```

Because the exact command names can vary between MAUI preview versions, use `--help` to confirm syntax before running the commands.

## 9. Confirm installations and platform support

Validate the environment before creating the application:

```bash
dotnet --version
dotnet --list-sdks
maui --version
dotnet workload list
```

You should see:

- .NET 11 preview SDK installed
- MAUI CLI prerelease installed
- MAUI workload installed
- platform tooling available for the host OS

## 10. Create the MAUI app folder and project

Create a folder for your MAUI application:

```bash
mkdir FirstMauiApp
cd FirstMauiApp
```

Create a `global.json` file in that folder if you have not already created one:

```json
{
  "sdk": {
    "version": "11.0.100-preview.7.26381.103",
    "allowPrerelease": true,
    "rollForward": "latestMinor"
  }
}
```

Now create the MAUI app using the .NET 11 SDK:

```bash
dotnet new maui -o FirstMauiApp
```

If you already created the folder and are in that folder, use:

```bash
dotnet new maui
```

## 11. Configure TargetFrameworks and RuntimeIdentifiers in the project

The project must target the correct MAUI TFMs for your operating system. Use the supported combinations below.

### Windows target configuration

For Windows, target Android and Windows together:

```xml
<PropertyGroup>
  <TargetFrameworks>net11.0-android;net11.0-windows10.0.19041.0</TargetFrameworks>
  <RuntimeIdentifiers>android-arm;android-arm64;win-x64;win-arm64</RuntimeIdentifiers>
</PropertyGroup>
```

Notes:

- Windows builds require the Windows target framework and Windows SDK support.
- Android emulator testing also remains available on Windows.

### macOS target configuration

For macOS, target Android, iOS, and Mac Catalyst:

```xml
<PropertyGroup>
  <TargetFrameworks>net11.0-android;net11.0-ios;net11.0-maccatalyst</TargetFrameworks>
  <RuntimeIdentifiers>android-arm;android-arm64;ios-arm64;iossimulator-x64;iossimulator-arm64;maccatalyst-x64;maccatalyst-arm64</RuntimeIdentifiers>
</PropertyGroup>
```

Notes:

- iOS requires the Apple toolchain and a valid signing configuration when deploying to a physical device.
- The simulator RID names can vary slightly depending on the SDK version, so validate using `dotnet build` and the MAUI tooling if needed.

### Linux target configuration

For Linux, target Android only:

```xml
<PropertyGroup>
  <TargetFrameworks>net11.0-android</TargetFrameworks>
  <RuntimeIdentifiers>android-arm;android-arm64</RuntimeIdentifiers>
</PropertyGroup>
```

Notes:

- Linux is for Android development support only.
- Linux is not a MAUI target platform for app packaging.

### Supported runtime IDs

For MAUI app development, the main runtime IDs to remember are:

- Windows: `win-x64`, `win-arm64`
- Android: `android-arm`, `android-arm64`
- iOS device: `ios-arm`, `ios-arm64`
- iOS simulator: `iossimulator-x64`, `iossimulator-arm64`
- Mac Catalyst: `maccatalyst-x64`, `maccatalyst-arm64`

## 12. Build the MAUI app

Change into the project directory and build:

```bash
cd FirstMauiApp
dotnet build
```

If the app targets multiple frameworks, you can build a specific target by specifying the TFM:

```bash
dotnet build -f net11.0-android
dotnet build -f net11.0-windows10.0.19041.0
dotnet build -f net11.0-ios
dotnet build -f net11.0-maccatalyst
```

Use `dotnet workload list` and `maui doctor` as needed if the build reports missing workloads or toolchain items.

## 13. Run the app on emulators, simulators, devices, and local workstations

### Run on Android emulator

```bash
dotnet build -f net11.0-android
dotnet build -f net11.0-android -t:Install -p:AndroidSdkDirectory=$ANDROID_SDK_ROOT
```

Then run from Visual Studio or use the MAUI command-line flow depending on the CLI preview version. In most cases, launching the app from VS Code or Visual Studio is the easiest method once the emulator is available.

### Run on iOS simulator (macOS only)

```bash
dotnet build -f net11.0-ios
```

Then launch the project from the Xcode/MAUI tooling or use the simulator from the IDE.

### Run on a physical device

For Android:

- Enable Developer Options
- Enable USB debugging
- Connect the device
- Ensure the device is authorized in adb
- Then build and run the Android target

For iOS (macOS only):

- Connect the iPhone or iPad
- Trust the machine in the device
- Use a valid Apple Developer signing configuration
- Build and run the iOS target from Visual Studio for Mac or Xcode-integrated tooling

### Run on the local workstation

On Windows, build and launch the Windows target:

```bash
dotnet build -f net11.0-windows10.0.19041.0
```

On macOS, you can run the Mac Catalyst target or test the iOS simulator if you are on a supported Apple development machine.

## 14. Recommended validation checklist

Before moving on to development work, verify the following:

- .NET 11 preview SDK is the active SDK
- `maui --version` returns a prerelease version
- `maui doctor` reports no critical issues
- `dotnet workload list` includes the MAUI preview workload
- Android emulator is created and starts successfully
- macOS simulator is created and starts successfully on Apple hardware
- A sample MAUI app builds successfully
- The app runs on at least one valid target (emulator, simulator, or local workstation)

## 15. Final development setup summary

The minimum environment for this workflow is:

- .NET 11 preview SDK
- MAUI CLI prerelease
- MAUI workload installed with previews enabled
- Visual Studio 2026 / VS Code with MAUI extension support
- Android emulator support
- macOS-specific Xcode tools for iOS and Mac Catalyst work
- Windows-specific C++ and Windows target support for Windows deployment

At that point, the project should be ready for MAUI app development using the preview toolchain.

## 16. Minimal command cheat sheet

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
code --profile mobile-dev --install-extension ms-dotnettools.dotnet-maui

mkdir FirstMauiApp
cd FirstMauiApp

cat > global.json <<'EOF'
{
  "sdk": {
    "version": "11.0.100-preview.7.26381.103",
    "allowPrerelease": true,
    "rollForward": "latestMinor"
  }
}
EOF

dotnet new maui -o FirstMauiApp
cd FirstMauiApp
dotnet build
```

This is the practical starting point for MAUI development on the latest .NET 11 preview with prerelease MAUI tooling.
