---
description: How to install a different Unity version
title: Installing a different Unity version
weight: 13
---

Each build machine image has a specific version of Unity installed. You can find out the specific Unity version by consulting the build machine specification in the documentation.

If you need to use a different Unity version, then it is possible to use the Unity Hub CLI to download and install a different Unity Editor version and target support files for that version. 

License activation and return takes place with the Unity version already installed on the machine, but building of the Xcode project or Android binary will use the version of Unity you install. 


## Getting the Unity version number and changeset id
In order to install a different version, you will need to know the full Unity version number and its **changeset** id. These details can be found in the Unity download archive [here](https://unity3d.com/get-unity/download/archive). To get the changeset id, first select the version you intend to use and then click on the ‘Release Notes’ button. Scroll to the bottom of the release notes and you will see the changeset id which you should make a note of. 


## Environment variables
You will need three environment variables in your codemagic.yaml called `UNITY_VERSION`, `UNITY_VERSION_CHANGESET`, and `UNITY_VERSION_BIN`. 

You should set the value of `UNITY_VERSION` to the full version number as displayed on the Unity Hub download archive page. For example, if the version is `2019.4.38f1`, set the variable as follows:

`UNITY_VERSION: 2019.4.38f1`

The value of `UNITY_VERSION_CHANGESET` should be set using the **changeset** id that you obtained from the release notes page on the Unity Hub download archive. For example, if the changeset id is `fdbb7325fa47`, then set the variable as follows:

`UNITY_VERSION_CHANGESET: fdbb7325fa47`

The `UNITY_VERSION_BIN` should be set as follows so the Unity binary path is declared for the version you want to build with:

Mac: `UNITY_VERSION_BIN: /Applications/Unity/Hub/Editor/${UNITY_VERSION}/Unity.app/Contents/MacOS/Unity`

Windows: `UNITY_VERSION_BIN: C:\Program Files\Unity\Hub\Editor\$UNITY_VERSION\Editor\Unity.exe`


## Activating Unity
Even though you are installing a different version of Unity to build your apps with, you should activate your license using the default Unity version already installed on the machine. Unity Hub CLI commands do not work correctly if a license is not already active on the machine.


## Unity installation script
After activating the Unity license as usual, add the following script to install the desired version and modules you wish to use. The example below uses Unity Hub CLI commands to install the specified Unity version as well as the Android and iOS Build Support modules.

Mac:

```yaml
    - name: Install Unity version
      script: | 
        /Applications/Unity\ Hub.app/Contents/MacOS/Unity\ Hub -- --headless install --version $UNITY_VERSION --changeset $UNITY_VERSION_CHANGESET 
        /Applications/Unity\ Hub.app/Contents/MacOS/Unity\ Hub -- --headless install-modules --version $UNITY_VERSION -m ios android 
```

Windows:

```yaml
    - name: Install Unity version
      script: | 
        New-Item ".\install-unity.bat" #create an empty batch file

        Set-Content install-unity.bat "`"$env:UNITY_HUB`" -- --headless install -v $env:UNITY_VERSION --changeset $env:UNITY_VERSION_CHANGESET"
        Add-Content install-unity.bat "`"$env:UNITY_HUB`" -- --headless install-modules --version $env:UNITY_VERSION -m ios android"

        Start-Process -FilePath ".\install-unity.bat" -Wait -NoNewWindow #start executing the batch file
```


## Building with the newly installed Unity version
Use the Unity version you installed on the machine:

```yaml
    - name: Build the Xcode project
      script: | 
        $UNITY_VERSION_BIN -batchmode -quit -logFile -projectPath . -executeMethod BuildScript.$BUILD_SCRIPT_IOS -nographics -buildTarget iOS
```

## Android Workflow configuration sample

Mac:

```yaml
workflows:
  unity-android-workflow:
      name: Unity Android Workflow
      instance_type: mac_pro
      max_build_duration: 120
      environment:
        groups:
          # Add the group environment variables in Codemagic UI (in Application or Team variables) - https://docs.codemagic.io/variables/environment-variable-groups/
          - unity # <-- (Includes UNITY_HOME, UNITY_SERIAL, UNITY_USERNAME and UNITY_PASSWORD)
          - google_play # <-- (Includes GCLOUD_SERVICE_ACCOUNT_CREDENTIALS <-- Put your google-services.json)
        vars:
          UNITY_BIN: $UNITY_HOME/Contents/MacOS/Unity
          UNITY_VERSION: 2019.4.38f1
          UNITY_VERSION_CHANGESET: fdbb7325fa47
          UNITY_VERSION_BIN: /Applications/Unity/Hub/Editor/${UNITY_VERSION}/Unity.app/Contents/MacOS/Unity
          BUILD_SCRIPT: BuildAndroid
          PACKAGE_NAME: "io.codemagic.unity" # <-- Put your package name here e.g. com.domain.myapp
        android_signing:
        - unity_test
        xcode: latest
      triggering:
        events:
          - push
          - tag
          - pull_request
        branch_patterns:
          - pattern: develop
            include: true
            source: true
      scripts:
        - name: Activate Unity License
          script: | 
            $UNITY_BIN -batchmode -quit -logFile -serial ${UNITY_SERIAL?} -username ${UNITY_USERNAME?} -password ${UNITY_PASSWORD?}      
        - name: Install Unity version, buld support modules, ndk and jdk
          script: | 
            /Applications/Unity\ Hub.app/Contents/MacOS/Unity\ Hub -- --headless install --version ${UNITY_VERSION} --changeset ${UNITY_VERSION_CHANGESET}
            /Applications/Unity\ Hub.app/Contents/MacOS/Unity\ Hub -- --headless install-modules --version ${UNITY_VERSION} -m android android-sdk-ndk-tools android-open-jdk          
        - name: Set build number and export Unity
          script: |
            export NEW_BUILD_NUMBER=$(($(google-play get-latest-build-number --package-name "$PACKAGE_NAME" --tracks=alpha) + 1))
            $UNITY_BIN_VERSION -batchmode -quit -logFile -projectPath . -executeMethod BuildScript.$BUILD_SCRIPT -nographics -buildTarget Android      
      artifacts:
        - android/*.aab
        - android/*.apk
      publishing:
        scripts:
          - name: Deactivate Unity License
            script: $UNITY_BIN -batchmode -quit -returnlicense -nographics
```
Windows:
```yaml
 unity-android-workflow:
    name: Unity Install Older Version Workflow
    max_build_duration: 120
    instance_type: windows_x2
    environment:
      groups:
        # Add the group environment variables in Codemagic UI (either in Application/Team variables) - https://docs.codemagic.io/variables/environment-variable-groups/
        - unity # <-- (Includes UNITY_HOME, UNITY_SERIAL, UNITY_USERNAME and UNITY_PASSWORD)
      vars:
        UNITY_BIN: $UNITY_HOME/Unity.exe
        UNITY_VERSION: 2021.3.3f1
        UNITY_VERSION_CHANGESET: af2e63e8f9bd
        UNITY_VERSION_BIN: C:\Program Files\Unity\Hub\Editor\$UNITY_VERSION\Editor\Unity.exe
        UNITY_HUB: C:\Program Files\Unity Hub\Unity Hub.exe
        BUILD_SCRIPT: BuildAndroid
        PACKAGE_NAME: "io.codemagic.unity" # <-- Put your package name here e.g. com.domain.myapp
      android_signing:
        - unity_test
    triggering:
      events:
        - push
      branch_patterns:
        - pattern: "*"
          include: true
      cancel_previous_builds: false
    scripts:
      - name: Activate Unity License (installed version)
        script: |
          cmd.exe /c "$env:UNITY_BIN" -batchmode -serial $env:UNITY_SERIAL -username $env:UNITY_USERNAME -password $env:UNITY_PASSWORD -quit -nographics
      - name: Install Unity version
        script: |
          New-Item ".\install-unity.bat" #create an empty batch file

          Add-Content install-unity.bat "`"$env:UNITY_HUB`" -- --headless install -v $env:UNITY_VERSION --changeset $env:UNITY_VERSION_CHANGESET"
          Add-Content install-unity.bat "`"$env:UNITY_HUB`" -- --headless install-modules --version $env:UNITY_VERSION -m android android-sdk-ndk-tools android-open-jdk"

          Start-Process -FilePath ".\install-unity.bat" -Wait -NoNewWindow #start executing the batch file
      - name: Build Unity Using (installed version)
        script: |
          cmd.exe /c "$env:UNITY_VERSION_BIN" -batchmode -quit -logFile "$env:CM_BUILD_DIR\\android\\log.txt" -projectPath . -executeMethod BuildScript.$env:BUILD_SCRIPT -nographics
    artifacts:
      - android/*.aab
      - android/*.apk
      - android/*.txt
    publishing:
      scripts:
        - name: Deactivate new Unity License using a Command Prompt
          script: |
            cmd.exe /c "$env:UNITY_BIN" -batchmode -quit -returnlicense -nographics 
```
