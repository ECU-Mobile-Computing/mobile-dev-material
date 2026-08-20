# Flutter Development: Windows Setup Instructions

### Install Android Studio - Google forces Android Studio to be installed even though it is not needed
1. Go to https://developer.android.com/studio
2. Download and run the installer for your OS.
3. Open Android Studio after installation.
4. During first launch, choose "Standard" setup and install all recommended components.
5. Installer should recognize that SDK has already been installed and not install again.

### Install Google Chrome Browser - Google forces Chrome to be installed because Flutter uses Chrome's debugging tools
> If you already have Chrome installed, then you should be able to skip this step
1. Go to https://www.google.com/chrome/dr/download
2. Click [Download Chrome]
3. Find the ChromeInstaller.exe that downloaded and execute it
4. Choose your own preferences during the install process
5. Chrome is not required to be you default browser

### Install VS Code
> If you already have VS Code installed, then you can skip this step...but you must make sure it is updated
1. Download VS Code installer  https://go.microsoft.com/fwlink/?LinkID=534107
2. Execute the downloaded installer: SSCodeUserSetup-x64-1.xxx.exe
3. Accept the defaults
4. Create a Flutter profile
5. Activate the Fl profile
6. Add the Flutter extension

### Install Flutter
1. Download the Flutter SDK zip 
  - https://storage.googleapis.com/flutter_infra_release/releases/stable/windows/flutter_windows_3.35.2-stable.zip
2. Extract the contents of the zip file to the newly created Flutter folder
```powershell
Expand-Archive `
      –Path $env:USERPROFILE\Downloads\flutter_windows_3.35.2-stable.zip `
      -Destination C:\SDK
```
3. Add Flutter binaries to the PATH
```powershell
$env:PATH += ";C:\SDK\flutter\bin
```
4. Reload profile
```powershell
. $PROFILE
```
5. Tell Flutter where to find the Android SDK
```powershell
 flutter config --android-sdk "C:\SDK\android"
```
6. Run `flutter doctor` to see if you environment is missing any Flutter requirements
```powershell
flutter doctor
```
### Verify Flutter installation
```powershell
flutter --version
```
### Run flutter doctor command and fix any reported issues
```powershell
flutter doctor
```
### Confirm that you can create and run a Flutter mobile app
1. Make sure an emulator is running
```powershell
emulator -avd my_pixel_35
```
2. Use File Explorer to navigate to your desired folder to place Flutter projects
3. Open terminal at this location
4. Create a new Flutter project
```powershell
flutter create hello_world
```
5. Navigate to the newly created project
```powershell
cd hello_world
```
6. Run the Flutter application
```powershell
flutter run
```
7. You should see a simple Hello World app appear in your emulator