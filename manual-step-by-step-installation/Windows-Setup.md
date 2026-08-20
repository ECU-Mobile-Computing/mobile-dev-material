# Android Mobile Development: Windows Setup Instructions

> **Note:** These instructions are to prepare a Windows computer for Android development.

> **Note:** Use PowerShell for all command line steps

> **Note:** Run PowerShell as Administrator unless a command specifically tells you not to run as Administrator.  For those commands only, open a new PowerShell and execute the commands.

> **Note:** Anytime you update environment variables, you must re-open the terminal or reload your profile before the change is visible to the terminal.


### Install Git
Git is required for version control and downloading code.

1. Go to https://git-scm.com/download/win
2. Download and run the installer. 
   - Make sure "Add Git to the PATH" is selected during install.
   - If you do not plan on using Git from the command line, then just accept all default options.
   - If you are going to use Git from the command line, then make sure you specify the terminal and text editor you want to use.
3. Open a new terminal and execute `git --version`
4. If it returns a version, then git installed successfully.
5. Make sure git is configured to make commits. 
   - Copy/paste `git config` commands below into Notepad
   - Leaving the double quotes, replace `Your Name` and `your.email@example.com` with your information
   - Execute the updated commands in a terminal. 
```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```
6. Confirm git configuration was updated successfully. Execute the command below in a terminal.  You should see your name and email address in the configuration.
```bash
git config --global --list
```

### Install Windows Package Installer (winget)
1. Check to see if winget is installed already
```powershell
winget --version
```
2. If not installed, navigate to https://apps.microsoft.com/detail/9nblggh4nns1?hl=en-US&gl=US
3. Click [Install]
4. Confirm installation
```powershell
winget --version
```



### Install Node.js (Required for React Native and some tools)
1. Check to see if node and npm are installed
```powershell
node -v
npm -v
```
2. Install Node
```powershell
winget install --id OpenJS.NodeJS -e
```
3. Confirm installation
```powershell
node -v
npm -v
```

### Install OpenJDK
Required for Android development.
1. Check to see if jdk is installed
```powershell
javac -version
```
2. Install jdk with winget 
```powershell
winget install --id Microsoft.OpenJDK.21 -e
```
3. Confirm installation
```powershell
javac --version
```

---

## 2. Android Development Setup

#### Install Android Studio Command line tools

1. Download the "Command line tools only" zip
  - Open https://developer.android.com/studio#cmdline-tools.
  - Scroll to the bottom of the web page
  - Click `commandlinetools-win-xxxxxxx_latest.zip` link
  - Agree to the Terms and Conditions
  - click [Download Android Command Line Tools for Windows]
  - Extract All from the downloaded zip file
  - The extracted folder should contain a cmdline-tools subfolder
2. Create a folder on your C: drive named `SDK` on your computer.  e.g. `C:\SDK`
```powershell
New-Item -Path "C:\SDK" -ItemType Directory
```
3. Create a folder inside `SDK` named `Android`. e.g. `C:\SDK\Android`
```powershell
New-Item -Path "C:\SDK\Android" -ItemType Directory
```
4. Create a folder inside `Android` named `cmdline-tools`.   e.g. `C:\SDK\Android\cmdline-tools`
```powershell
New-Item -Path "C:\SDK\Android\cmdline-tools" -ItemType Directory
```
5. Copy the whole `cmdline-tools` folder from Downloads to the `C:\SDK\Android\cmdline-tools` folder.  e.g.`C:\SDK\Android\cmdline-tools\cmdline-tools` 
6. Rename the nested `cmdline-tools` folder latest
```powershell
Rename-Item -Path "C:\SDK\Android\cmdline-tools\cmdline-tools" -NewName "C:\SDK\Android\cmdline-tools\latest"
```
7. Open PowerShell as Administrator.
8. Set environment variables:
   ```powershell
   $env:ANDROID_HOME = "C:\SDK\Android"   # Use the path where you extracted the SDK. Substitute your own if different.
   $env:PATH += ";$env:ANDROID_HOME\cmdline-tools\latest\bin;$env:ANDROID_HOME\platform-tools;$env:ANDROID_HOME\emulator"
   ```
9. Navigate to the cmdline-tools' bin directory:
   ```powershell
   Set-Location "C:\SDK\Android\cmdline-tools\latest\bin"   # Use your actual path if different.
   ```
10. Accept licenses and install Android packages:
   ```powershell
   ./sdkmanager --licenses                # Accept all SDK licenses
   ./sdkmanager "platform-tools"          # Install platform tools (adb, fastboot, etc.)
   ./sdkmanager "platforms;android-35"    # Install Android 13 (API 35) platform
   ./sdkmanager "platforms;android-36"    # Install Android 14 (API 36) platform
   ./sdkmanager "build-tools;35.0.0"      # Install build tools for API 35
   ./sdkmanager "build-tools;36.0.0"      # Install build tools for API 36
   ./sdkmanager "emulator"                # Install Android emulator software
   ```
  
  
  > If you are using an Intel or AMD process, then use the below commands.

  ```powershell 
./sdkmanager "system-images;android-35;google_apis;x86_64" # Install emulator image for API 35
./sdkmanager "system-images;android-36;google_apis;x86_64" # Install emulator image for API 36
  ```
  
  > If you are using an ARM processor (Snap Dragon, etc...), then use the below commands.
  
```powershell 
./sdkmanager "system-images;android-35;google_apis;arm64-v8a" # Install emulator image for API 35
./sdkmanager "system-images;android-36;google_apis;arm64-v8a" # Install emulator image for API 36
```

11. Confirm pixel_9 device definition is available
```powershell
./avdmanager list device
```
12. Create emulators - based on your CPU architecture select either x86_64 or arm64-v8a:

  If you are using an Intel or AMD process, then use the below commands.
  ```powershell
  ./avdmanager create avd --name my_pixel_35 --device pixel_9 --package "system-images;android-35;google_apis;x86_64" # Create emulator for API 35
  ./avdmanager create avd --name my_pixel_36 --device pixel_9 --package "system-images;android-36;google_apis;x86_64" # Create emulator for API 36
  ```
  If you are using an ARM processor (Snap Dragon, etc...), then use the below commands.
  ```powershell
  ./avdmanager create avd --name my_pixel_35 --device pixel_9 --package "system-images;android-35;google_apis;arm64-v8a" # Create emulator for API 35
  ./avdmanager create avd --name my_pixel_36 --device pixel_9 --package "system-images;android-36;google_apis;arm64-v8a" # Create emulator for API 36
  ```
13. Start one of your emulators to confirm it works
  Allow access through the firewall when prompted
```powershell
    emulator -avd my_pixel_35
```
---

### Make Android SDK Tools Globally Accessible

After installing the command line tools, make the common binaries like sdkmanager, avdmanager, adb, and emulator available in both your terminal and command prompt from any directory.

#### Windows (PowerShell)
Add the following to your user or system PATH environment variable (if not already set):
```powershell
$env:PATH += ";C:\SDK\Android\cmdline-tools\latest\bin;C:\SDK\Android\platform-tools"
```

Create PowerShell aliases in your profile for convenience:
```powershell
Set-Alias sdkmanager "C:\SDK\Android\cmdline-tools\latest\bin\sdkmanager.bat"
Set-Alias avdmanager "C:\SDK\Android\cmdline-tools\latest\bin\avdmanager.bat"
Set-Alias adb "C:\SDK\Android\platform-tools\adb.exe"
Set-Alias emulator "C:\SDK\Android\emulator\emulator.exe"

# 1. Ensure profile file exists
if (-not (Test-Path -LiteralPath $PROFILE)) {
  New-Item -ItemType File -Path $PROFILE -Force | Out-Null
}

# 2. Append aliases (only if not already present)
$aliases = @'
Set-Alias sdkmanager "C:\SDK\Android\cmdline-tools\latest\bin\sdkmanager.bat"
Set-Alias avdmanager "C:\SDK\Android\cmdline-tools\latest\bin\avdmanager.bat"
Set-Alias adb        "C:\SDK\Android\platform-tools\adb.exe"
Set-Alias emulator   "C:\SDK\Android\emulator\emulator.exe"
'@

$profileContent = Get-Content -LiteralPath $PROFILE -Raw
  Add-Content -LiteralPath $PROFILE -Value "`n# Android CLI aliases added $(Get-Date -Format 'u')`n$aliases"

# 3. Reload profile
. $PROFILE

# 4. Verify
Get-Command sdkmanager, avdmanager, adb, emulator

```

