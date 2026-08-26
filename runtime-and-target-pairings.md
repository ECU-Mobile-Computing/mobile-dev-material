# MAUI Target Framework + Runtime ID Pairings

## Android

| TFM | Device RIDs | Emulator RIDs | Architectures | AOT Support |
| --- | --- | --- | --- | --- |
| **net6.0-android** | android-arm, android-arm64 | android-x86, android-x64 | ARM32, ARM64, x86, x64 | Yes (limited) |
| **net7.0-android** | android-arm, android-arm64 | android-x86, android-x64 | ARM32, ARM64, x86, x64 | Yes |
| **net8.0-android** | android-arm, android-arm64 | android-x86, android-x64 | ARM32, ARM64, x86, x64 | Yes |
| **net9.0-android** | android-arm, android-arm64 | android-x86, android-x64 | ARM32, ARM64, x86, x64 | Yes |
| **net10.0-android** | android-arm, android-arm64 | android-x86, android-x64 | ARM32, ARM64, x86, x64 | Yes |
| **net11.0-android** | android-arm, android-arm64 | android-x86, android-x64 | ARM32, ARM64, x86, x64 | Yes |

## iOS

| TFM | Device RIDs | Simulator RIDs | Architectures | AOT Support |
| --- | --- | --- | --- | --- |
| **net6.0-ios** | ios-arm, ios-arm64 | iossimulator-x86, iossimulator-x64 | ARM32, ARM64, x86, x64 | Required (iOS always AOT) |
| **net7.0-ios** | ios-arm, ios-arm64 | iossimulator-x86, iossimulator-x64 | ARM32, ARM64, x86, x64 | Required |
| **net8.0-ios** | ios-arm, ios-arm64 | iossimulator-x86, iossimulator-x64 | ARM32, ARM64, x86, x64 | Required |
| **net9.0-ios** | ios-arm, ios-arm64 | iossimulator-x86, iossimulator-x64 | ARM32, ARM64, x86, x64 | Required |
| **net10.0-ios** | ios-arm, ios-arm64 | iossimulator-x86, iossimulator-x64 | ARM32, ARM64, x86, x64 | Required |
| **net11.0-ios** | ios-arm, ios-arm64 | iossimulator-x86, iossimulator-x64 | ARM32, ARM64, x86, x64 | Required |


## Mac Catalyst

| TFM | RIDs | Architectures | AOT Support |
| --- | --- | --- | --- |
| **net6.0-maccatalyst** | maccatalyst-x64, maccatalyst-arm64 | Intel, Apple Silicon | Optional |
| **net7.0-maccatalyst** | maccatalyst-x64, maccatalyst-arm64 | Intel, Apple Silicon | Optional |
| **net8.0-maccatalyst** | maccatalyst-x64, maccatalyst-arm64 | Intel, Apple Silicon | Optional |
| **net9.0-maccatalyst** | maccatalyst-x64, maccatalyst-arm64 | Intel, Apple Silicon | Optional |
| **net10.0-maccatalyst** | maccatalyst-x64, maccatalyst-arm64 | Intel, Apple Silicon | Optional |

## Windows

| TFM | RIDs | Architectures | AOT Support |
| --- | --- | --- | --- |
| **net6.0-windows10.0.19041.0** | win-x64, win-arm64 | x64, ARM64 | Optional |
| **net7.0-windows10.0.19041.0** | win-x64, win-arm64 | x64, ARM64 | Optional |
| **net8.0-windows10.0.19041.0** | win-x64, win-arm64 | x64, ARM64 | Optional |
| **net9.0-windows10.0.19041.0** | win-x64, win-arm64 | x64, ARM64 | Optional |
| **net10.0-windows10.0.19041.0** | win-x64, win-arm64 | x64, ARM64 | Optional |

## Linux

As of 8/20/2026, you can develop MAUI apps on Linux, but you cannot target Linux.  Linux does have .net Target Frameworks and Runtime IDs, but they cannot be used for MAUI.

NOT SUPPORTED BY MAUI

| TFM | RIDs | Architectures | 
| --- | --- | --- | 
|net8.0|linux-x64, linux-arm64, linux-musl-x64, linux-musl-arm64|x64, ARM64|
|net9.0|linux-x64, linux-arm64, linux-musl-x64, linux-musl-arm64|x64, ARM64|
|net10.0|linux-x64, linux-arm64, linux-musl-x64, linux-musl-arm64|x64, ARM64|
