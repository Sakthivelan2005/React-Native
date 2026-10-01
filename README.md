Setting up a React Native CLI environment on Windows requires strictly configuring Node, Java, and specific Android SDK versions. This process avoids Expo entirely and compiles native code directly.

1. **Install Node.js and JDK:** Use Chocolatey to manage dependencies.
React Native requires Node.js and a Java Development Kit (JDK 17 is the current standard). Open an **Administrator Command Prompt** and run:

```bash
choco install -y nodejs-lts microsoft-openjdk17

```


2. **Install Android Studio:** The base IDE and emulators.
Download and install [Android Studio](https://developer.android.com/studio). During the installation wizard, ensure all of the following are checked:

* Android SDK
* Android SDK Platform
* Android Virtual Device
* Performance (Intel HAXM) or Hyper-V (if prompted)


3. **Configure the Android SDK:** Specific SDK versions are strictly required.
React Native requires the **Android 15 (VanillaIceCream) SDK**.

1. Open Android Studio, click **More Actions**, and select **SDK Manager**.
2. Under the **SDK Platforms** tab, check the box for **Show Package Details** (bottom right).
3. Expand **Android 15 (VanillaIceCream)** and check:
* `Android SDK Platform 35`
* `Intel x86 Atom_64 System Image`


4. Switch to the **SDK Tools** tab, check **Show Package Details** again, and ensure these are selected:
* `36.0.0` (under Android SDK Build-Tools)
* `Android SDK Command-line Tools (latest)`


5. Click **Apply** to download and install.


4. **Set Environment Variables:** Crucial for the CLI to locate Android build tools.
The React Native CLI requires the `ANDROID_HOME` environment variable to build apps.

1. Open the Windows **Control Panel**, select **User Accounts**, and click **Change my environment variables**.
2. Click **New...** under **User variables** and add:
* **Variable name:** `ANDROID_HOME`
* **Variable value:** `%LOCALAPPDATA%\Android\Sdk`


3. Select the **Path** variable under User variables, click **Edit**, and add these two new paths:
* `%LOCALAPPDATA%\Android\Sdk\platform-tools`
* `%LOCALAPPDATA%\Android\Sdk\emulator`




5. **Initialize and Run Your Project:** Create the React Native CLI app.
Open a normal Command Prompt (not Administrator) in the folder where you want your project, and run:

```bash
npx react-native@latest init MyAwesomeApp

```

Once the project is created, navigate into the folder and start the Android build:

```bash
cd MyAwesomeApp
npm run android

```

This command launches the Metro bundler and starts the app on your Android Virtual Device (AVD).