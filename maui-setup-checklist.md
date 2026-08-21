# MAUI Setup Checklist

## Prerequisites

- [ ] Install the latest .NET 11 preview SDK
- [ ] Confirm `dotnet --version` shows the preview
- [ ] Confirm `dotnet --list-sdks` includes .NET 11

## Install MAUI tools

- [ ] Install the prerelease MAUI CLI: `dotnet tool install -g microsoft.maui.cli --prerelease`
- [ ] Verify: `maui --version`
- [ ] Run: `maui doctor`
- [ ] Run: `maui doctor --fix` if issues are reported
- [ ] Install the MAUI workload: `dotnet workload install maui --include-previews`
- [ ] Verify: `dotnet workload list`

## VS Code setup

- [ ] Install VS Code
- [ ] Create profile: `mobile-dev`
- [ ] Activate the `mobile-dev` profile
- [ ] Enable `NuGet: Allow Prerelease Versions`
- [ ] Install required MAUI extensions
- [ ] Enable the MCP marketplace
- [ ] Add Microsoft Learn and Microsoft NuGet MCP servers
- [ ] Restart VS Code

## Platform setup

### Windows

- [ ] Install Visual Studio 2026 Community
- [ ] Select MAUI development workload
- [ ] Select Windows C++ development workload
- [ ] Install required Windows target support and runtime support

### macOS

- [ ] Install or update Xcode
- [ ] Accept Xcode license: `sudo xcodebuild -license accept`
- [ ] Ensure command line tools are installed

### Linux

- [ ] Install the .NET 11 preview SDK
- [ ] Install the MAUI workload
- [ ] Prepare Android tooling only

## Emulator and simulator

- [ ] Create Android emulator using MAUI CLI
- [ ] Start Android emulator
- [ ] Confirm emulator is available
- [ ] On macOS, create and run iOS simulator

## Create and verify the app

- [ ] Create a folder for the project
- [ ] Add `global.json` with the .NET 11 preview SDK pin
- [ ] Create the MAUI app: `dotnet new maui`
- [ ] Configure TargetFrameworks and RuntimeIdentifiers for the platform
- [ ] Run: `dotnet build`
- [ ] Build the Android, Windows, iOS, or Mac Catalyst target as needed
- [ ] Run the app on the emulator, simulator, device, or local workstation

## Final validation

- [ ] `maui --version` succeeds
- [ ] `maui doctor` shows no critical issues
- [ ] `dotnet workload list` includes MAUI preview workloads
- [ ] The app builds successfully
- [ ] The app runs successfully on at least one supported target

## Command reference

```bash
dotnet --version
dotnet --list-sdks

dotnet tool install -g microsoft.maui.cli --prerelease
maui --version
maui doctor
maui doctor --fix

dotnet workload install maui --include-previews
dotnet workload list

dotnet new maui
dotnet build
```
