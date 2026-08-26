# MAUI Target Runtimes

| Platform | Valid MAUI TFMs |
| --- | --- |
| **Android** | net6.0‑android → net10.0‑android |
| **iOS** | net6.0‑ios → net10.0‑ios |
| **Mac Catalyst** | net6.0‑maccatalyst → net10.0‑maccatalyst |
| **Windows 11** | net10.0‑windows10.0.22621.0 → net10.0‑windows10.0.28000.2114 |


Target Runtimes are only available if they have been installed.
```bash
# you can install all possible maui workloads with this command
dotnet workload install maui
# or you can install specific maui workloads
dotnet workload install maui-android
dotnet workload install maui-ios
dotnet workload install maui-maccatalyst
dotnet workload install maui-windows
dotnet workload install maui-tizen
dotnet workload install maui-desktop
```

.Net has many possible workloads.  Not all workloads are compatible with MAUI.  MAUI workloads are all prefixed with `maui-` 
```bash
dotnet workload list   # shows installed workloads
dotnet workload search # shows available workloads
```

These are the possible Windows 11 Target Runtimes.
net10.0‑windows10.0.22621.0
net10.0‑windows10.0.26100.8249
net10.0‑windows10.0.28000.2114

In order to target one of these Windows Target Runtimes, you must first install the maui-windows workload.
```bash
dotnet workload install maui # installs all Target Runtimes
# or
dotnet workload install maui-windows # installs the MAUI Windows target frameworks
```

You must also have the `matching` Windows SDK installed using the Visual Studio Installer.
```
Windows 11 SDK (10.0.22621.0)
Windows 11 SDK (10.0.26100.8249)
Windows 11 SDK (10.0.28000.2114)
```
