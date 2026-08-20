# Flutter Development: Mac Setup Instructions

### If using Mac M1-M4 install rosetta - this allows you to run Intel-only apps
```ZSH
sudo softwareupdate --install-rosetta --agree-to-license
```

### Install CocoaPods - used by Flutter for integrating native iOS dependencies
```zsh
sudo gem install cocoapods
```

### Install Android Studio - Google forces Android Studio to be installed even though it is not needed
1. Go to https://developer.android.com/studio
2. Click [Download Android Studio Narwhal Feature Drop]
3. Check the read and agree checkbox
4. Download and run the installer for your OS.
   1. If you have an M1, M2, M3, M4, etc, then click [Mac with Apple chip]
   2. Otherwise; click [Mac with Intel chip]
5. Double click downloaded .dmg file and install Android Studio
6. Open Android Studio after installation.
7. If prompted to update, then apply the updates
8. During first launch, choose "Standard" setup and install all recommended components.
9. Installer should recognize that SDK has already been installed and not install again.

### Install Google Chrome Browser - Google forces Chrome to be installed because Flutter uses Chrome's debugging tools
> If you already have Chrome installed, then you should be able to skip this step
> Note: Chrome is not required to be you default browser
1. Go to https://www.google.com/chrome/dr/download
2. Click [Download Chrome]
3. Double click the downloaded .dmg file
4. Install Chrome into the Applications folder

### Install VS Code
> If you already have VS Code installed, then you can skip this step...but you must make sure it is updated
1. Navigate to https://code.visualstudio.com/download
2. Click the Mac download option
3. Install Visual Studio Code.app into the Applications folder
4. Open Visual Studio Code

### Create a VS Code Flutter profile
1. Click on the cog icon in the bottom left in vs code window
2. Select profile --> profiles
3. Click [New Profile]
4. Type `Flutter` for profile name
5. Click [Create]
6. Click the checkmark next to the newly added Flutter profile to activate it
   
### Add required extensions to Flutter profile
1. Click the Extensions tab on the Activity bar in vs code (same bar as the cog)
2. Search for `Flutter`.  The publisher is Dart Code.
3. Install `Flutter`.  This should also install the `Dart` extension.

> Note: When you open vs code, it will probably open with a profile named `Default`.  Always, make sure you select your Flutter profile when doing Flutter development.

> Note: Flutter install page - https://docs.flutter.dev/get-started/install/macos

### Download Flutter SDK zip file
1. Download Flutter SDK

If you have a `Silicon Mac`, then download this Flutter SDK

https://storage.googleapis.com/flutter_infra_release/releases/stable/macos/flutter_macos_arm64_3.35.2-stable.zip

If you have an `Intel Mac`, then download this Flutter SDK

https://storage.googleapis.com/flutter_infra_release/releases/stable/macos/
flutter_macos_3.35.2-stable.zip

### Install Flutter SDK
1. Create Development folder in home directory and extract contents of Flutter SDK into Development folder
> Note: This assumes you have not moved your default downloads location
```ZSH
mkdir -p ~/Development
unzip ~/Downloads/flutter_macos_*.zip -d ~/Development
```
2. Add Flutter to the PATH
```ZSH
echo 'export PATH="$PATH:$HOME/Development/flutter/bin"' >> ~/.zprofile
```
3. Reload your profile
```ZSH
source ~/.zprofile  
```
### Verify Flutter installation
```ZSH
flutter --version
```
### Run flutter doctor command and fix any reported issues
```ZSH
flutter doctor
```
### Confirm that you can create and run a Flutter mobile app
1. Start emulator if it is not currently running
```ZSH
emulator -avd my_pixel_35
```
2. Navigate to where you want to create your Flutter projects
3. Create a new Flutter project
```ZSH
flutter create hello_world
```
4. Navigate to the newly created project
```ZSH
cd hello_world
```
5. Run the Flutter application
```ZSH
flutter run
```
6. If prompted for which device you want to use, then select the running emulator...unless you setup a physical device
7. It will take a little time to compile and deploy. Especially, the first time.
8. You should see a simple Hello World app appear in your emulator