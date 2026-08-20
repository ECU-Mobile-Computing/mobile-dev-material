# Android Mobile Development: Mac Setup Instructions

> **Note:** These instructions are to prepare a Mac computer for android development.

> **Note:** Use `ZSH` for all command line steps

> **Note:** Anytime you update environment variables, you must re-open the terminal or reload your profile before the change is visible to the terminal.

### Make sure Homebrew is installed
1. Run command below in terminal
```ZSH
brew --version
```
2. If brew command is not recognized then install Homebrew
```ZSH
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
3. Add brew command to the PATH
```ZSH
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```
5. Verify brew was installed successfully
```ZSH
brew --version
```

### Install xcode
1. Check to see if xcode is installed.  If installed, then is it a recent version (16.4)?
```zsh
xcodebuild -version
```
2. If you have a recent version installed, skip to step 5. If you get a message saying no developer tools were found, then you need to continue with step 3.
3. Download latest version of `xcode` from App Store
4. Install downloaded `xcode`
5. Set latest version of `xcode` as the default version
```zsh
sudo sh -c 'xcode-select -s /Applications/Xcode.app/Contents/Developer && xcodebuild -runFirstLaunch'
```
6. Accept `xcode` license agreement
```zsh
sudo xcodebuild -license
# type agree
```

### Install Git
Git is required for version control and downloading code.
1. Open Terminal.
2. Run: `git --version`
3. If not installed, run: `xcode-select --install` and follow prompts.
4. Make sure git is configured to make commits. 
   - Copy/paste `git config` commands below into Notepad
   - Leaving the double quotes, replace `Your Name` and `your.email@example.com` with your information
   - Execute the updated commands in a terminal. 
```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```
5. Confirm git configuration was updated successfully. Execute the command below in a terminal.  You should see your name and email address in the configuration.
```bash
git config --global --list
```

### Install Node.js (Required for React Native and some tools)
1. Install node
```zsh
brew install node
```
2. Verify node and npm are installed
```
node -v
npm -v
```

### Install JDK 
1. Check to see if Java is installed
```ZSH
javac -version
```
2. If brew command is not recognized then install JDK
```ZSH
brew install openjdk
```
3. Link JDK so it is available where applications expect to find it
```ZSH
brew link --force --overwrite openjdk
```
4. Add JDK to the PATH in your profile
> this assumes you installed JDK with brew
```ZSH
echo 'export PATH="/opt/homebrew/opt/openjdk/bin:$PATH"' >> ~/.zprofile
```
 - Reload your profile
```ZSH
source ~/.zprofile  
```
 - Verify JDK is installed
```ZSH
javac -version
```

---

## 2. Android Development Setup

#### Install Android Studio Command line tools

1. Download the "Command line tools only" zip
  1. Open https://developer.android.com/studio#cmdline-tools.
  2. Scroll to the bottom of the web page
  3. Click commandlinetools-mac-13114758_latest.zip link
  4. Agree to the Terms and Conditions
  5. click [Download Android Command Line Tools for Mac]
  6. Manually extract All from the downloaded zip file
  7. The extracted folder should be named cmdline-tools
2. Create SDK folder and unzip the tools 
```ZSH
# Create cmdline-tools directory in the sdk folder
mkdir -p ~/Library/Development/Android/sdk/cmdline-tools
# Copy the manually extracted cmdline-tools folder from Downloads to newly created cmdline-tools in sdk folder
> this assumes you have not moved the downloads location
cp -R ~/Downloads/cmdline-tools ~/Library/Development/Android/sdk/cmdline-tools
# Rename the copied cmdline-tools folder to latest
mv ~/Library/Development/Android/sdk/cmdline-tools/cmdline-tools ~/Library/Development/Android/sdk/cmdline-tools/latest
```
1. Append Android path information to the end of your profile

```ZSH
echo 'export PATH="/opt/homebrew/opt/openjdk/bin:$PATH"' >> ~/.zprofile
echo 'export ANDROID_HOME="$HOME/Library/Development/Android/sdk"' >> ~/.zprofile
echo 'export ANDROID_SDK_ROOT="$ANDROID_HOME"' >> ~/.zprofile
echo 'export PATH="$PATH:$ANDROID_HOME/cmdline-tools/latest/bin"' >> ~/.zprofile
echo 'export PATH="$PATH:$ANDROID_HOME/platform-tools"' >> ~/.zprofile
echo 'export PATH="$PATH:$ANDROID_HOME/emulator"' >> ~/.zprofile
```
4. Reload your profile
```ZSH
source ~/.zprofile  
```
5. Accept licenses and install Android packages: 
> Note: You may have to copy this commands one line at a time without the comment #
   ```ZSH
   sdkmanager --licenses                # Type 'y' for each Accept?
   sdkmanager "platform-tools"          # Install platform tools (adb, fastboot, etc.)
   sdkmanager "platforms;android-35"    # Install Android 13 (API 35) platform
   sdkmanager "platforms;android-36"    # Install Android 14 (API 36) platform
   sdkmanager "build-tools;35.0.0"      # Install build tools for API 35
   sdkmanager "build-tools;36.0.0"      # Install build tools for API 36
   ```
If you `have` an Mac Silicon e.g. M1, M2, M3, M4, etc... use the following command
   ```zsh
   sdkmanager "system-images;android-35;google_apis;arm64-v8a" # Install emulator image for API 35 (arm64-v8a)
   sdkmanager "system-images;android-36;google_apis;arm64-v8a" # Install emulator image for API 36 (arm64-v8a)
   ```

If you `do not have` an Mac Silicon e.g. M1, M2, M3, M4, etc... use the following command
   ```zsh
   sdkmanager "system-images;android-35;google_apis;x86_64" # Install emulator image for API 35 (x86_64)
   sdkmanager "system-images;android-36;google_apis;x86_64" # Install emulator image for API 36 (x86_64)
   ```
1. Confirm pixel_9 device definition is available
   ```ZSH
   avdmanager list device
   ```
2.  Create emulators:
   
If you `have` an Mac Silicon e.g. M1, M2, M3, M4, etc... use the following command
   ```ZSH
   avdmanager create avd --name my_pixel_35 --device pixel_9 --package "system-images;android-35;google_apis;arm64-v8a" # Create emulator for API 35. note: x86_64 for targeting Intel Macs
   avdmanager create avd --name my_pixel_36 --device pixel_9 --package "system-images;android-36;google_apis;arm64-v8a" # Create emulator for API 36 note: x86_64 for targeting Intel Macs
   ```

If you `do not have` an Mac Silicon e.g. M1, M2, M3, M4, etc... use the following command
   ```ZSH
   avdmanager create avd --name my_pixel_35 --device pixel_9 --package "system-images;android-35;google_apis;x86_64" # Create emulator for API 35. note: x86_64 for targeting Intel Macs
   avdmanager create avd --name my_pixel_36 --device pixel_9 --package "system-images;android-36;google_apis;x86_64" # Create emulator for API 36 note: x86_64 for targeting Intel Macs
   ```
3.  Start one of your emulators to confirm it works
```ZSH
    emulator -avd my_pixel_35
```
---