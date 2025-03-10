---
title: Signing macOS apps
description: How to set up macOS code signing in codemagic.yaml
weight: 2
aliases: /code-signing-yaml/signing-macos
---

All macOS applications have to be digitally signed before they can be installed on devices or made available to the public via the Mac App Store or outside of the Mac App Store.

{{<notebox>}}
This guide only applies to workflows configured with the **codemagic.yaml**. If your workflow is configured with **Flutter workflow editor** please go to [Signing macOS apps using the Flutter workflow editor](../code-signing/macos-code-signing).
{{</notebox>}}

## Prerequisites


Signing macOS applications requires [Apple Developer Program](https://developer.apple.com/programs/enroll/) membership. 

Note that in order to publish to Mac App Store, the application must be signed with a `Mac App Distribution` certificate using a `Mac App Store` provisioning profile. Additionally, the application must be packaged into a `.pkg` Installer package which should be signed with a `Mac Installer Distribution` certificate.

Notarization for distributing the app outside the Mac App Store is not yet available.

You can upload your signing certificate and distribution profile to Codemagic to manage code signing yourself or use the automatic code signing option where Codemagic takes care of code signing and signing files management on your behalf. Read more about the two options below.


To set up publishing the code-signed application to App Store Connect, refer [here](../yaml-publishing/app-store-connect).

{{<notebox>}}
Under the hood, we use [Codemagic CLI tools](https://github.com/codemagic-ci-cd/cli-tools) to perform macOS code signing ⏤ these tools are open source and can also be [used locally](../cli/codemagic-cli-tools/) or in other environments. More specifically, we use the [xcode-project utility](https://github.com/codemagic-ci-cd/cli-tools/blob/master/docs/xcode-project/README.md) for preparing the code signing properties for the build, the [keychain utility](https://github.com/codemagic-ci-cd/cli-tools/blob/master/docs/keychain/README.md) for managing macOS keychains and certificates, and the [app-store-connect utility](https://github.com/codemagic-ci-cd/cli-tools/blob/master/docs/app-store-connect/README.md) for creating and downloading code signing certificates and provisioning profiles. The latter makes use of the App Store Connect API for authenticating with the Apple Developer Portal.
{{</notebox>}}

## Automatic code signing

In order to use automatic code signing and have Codemagic manage signing certificates and provisioning profiles on your behalf, you need to configure API access to App Store Connect.

### Creating the App Store Connect API key

{{< include "/partials/app-store-connect-api-key.md">}}

### Saving the API key to environment variables

Save the API key and the related information in the **Environment variables** section of the Codemagic UI and select **Secure**. Note that binary files (i.e. provisioning profiles & .p12 certificate) have to be [`base64 encoded`](../variables/environment-variable-groups/#storing-sensitive-valuesfiles) locally before they can be saved to **Environment variables** and decoded during the build. 

Below are the environment variables you need to set:

- `APP_STORE_CONNECT_KEY_IDENTIFIER`

  In **App Store Connect > Users and Access > Keys**, this is the **Key ID** of the key.

- `APP_STORE_CONNECT_ISSUER_ID`

  In **App Store Connect > Users and Access > Keys**, this is the **Issuer ID** displayed above the table of active keys.

- `APP_STORE_CONNECT_PRIVATE_KEY`

  This is the private API key downloaded from App Store Connect. On macOS, you can use `pbcopy < AuthKey_XXXXXX.p8` to copy the contents of the private key.

- `CERTIFICATE_PRIVATE_KEY`

  An RSA 2048 bit private key to be included in the [signing certificate](https://help.apple.com/xcode/mac/current/#/dev1c7c2c67d) that Codemagic fetches or creates. You'll need to either create a new certificate and private key or find an existing one. On macOS, you can use `pbcopy < private_key` to copy the contents of the private key.

  App Developer Portal has a limitation of maximum of 2 macOS distribution certificates per team. This means that if you already have 2 `Mac Installer Distribution` or `Developer ID Application` certificates, you won't be able to create new ones. If any of those are not used, you may revoke them in the [Apple Developer Portal](https://developer.apple.com/account/resources/certificates/list), which will make it possible to create new certificates with the specified new certificate private key. You can create a new 2048 bit RSA key by running the following command in your terminal:

  ```bash
  ssh-keygen -t rsa -b 2048 -m PEM -f ~/Desktop/codemagic_private_key -q -N ""
  ```

   If you wish to use existing certificates and don't have the private key used to create them available, or those certificates were created via a Certificate Authority request or via Xcode, you can [export](https://help.apple.com/xcode/mac/current/#/dev154b28f09) the existing certificate(s) into `.p12` container(s) and get the used private key to be able to fetch these certificates from another machine. You can export the private key and convert it using the following command:

    ```bash
    openssl pkcs12 -in <your_certificate_name>.p12 -nodes -nocerts | openssl rsa -out <your_private_key_name>.key
    ```

{{<notebox>}}
Tip: Store all the App Store Connect variables in the same group so they can be imported to codemagic.yaml workflow at once. 
  
If the group of variables is reusable for various applications, they can be defined in [Global variables and secrets](../variables/environment-variable-groups/#global-variables-and-secrets) in **Team settings** for easier access.
{{</notebox>}}

Add the group in the **codemagic.yaml** as follows:

```yaml
environment:
  groups:
    - app_store_key_properties
    - certificate
    
  # Add the above mentioned group environment variables in Codemagic UI (either in Application/Team variables): 
    # APP_STORE_CONNECT_ISSUER_ID
    # APP_STORE_CONNECT_KEY_IDENTIFIER
    # APP_STORE_CONNECT_PRIVATE_KEY
    # CERTIFICATE_PRIVATE_KEY
```

{{<notebox>}}
Alternatively, each property can be specified in the `scripts` section of the YAML file as a command argument to programs with dedicated flags. See the details [here](https://github.com/codemagic-ci-cd/cli-tools/blob/master/docs/app-store-connect/fetch-signing-files.md#--issuer-idissuer_id). In that case, the environment variables will be fallbacks for missing values in scripts.
{{</notebox>}}

### Adding steps to code sign the app

To code sign the app, add the following commands in the [`scripts`](../getting-started/yaml#scripts) section of the configuration file, after all the dependencies are installed, right before the build commands.

```yaml
scripts:
  ... your dependencies installation
  - name: Set up keychain to be used for codesigning using Codemagic CLI 'keychain' command
    script: keychain initialize
  - name: Fetch Mac App Distribution certificate and Mac App Store profile
    script: |
      # Allow creating resources if existing are not found with `--create` flag. You may omit this flag if you already have the required certificate and profile and provided the corresponding private key
      app-store-connect fetch-signing-files \
        "io.codemagic.app" \
        --platform MAC_OS \
        --type MAC_APP_STORE \
        --create
  - name: Fetch Mac Installer Distribution certificates
    script: |
      # You may omit the first command if you already have the installer certificate and provided the corresponding private key
      app-store-connect create-certificate --type MAC_INSTALLER_DISTRIBUTION --save || \
        app-store-connect list-certificates --type MAC_INSTALLER_DISTRIBUTION --save
  - name: Set up signing certificate
    script: keychain add-certificates
  - name: Set up code signing settings on Xcode project
    script: xcode-project use-profiles
  ... your build commands
```

Instead of specifying the exact bundle-id, you can use `"$(xcode-project detect-bundle-id)"`.

\* Based on the specified bundle ID and [provisioning profile type](https://github.com/codemagic-ci-cd/cli-tools/blob/master/docs/app-store-connect/fetch-signing-files.md#--typeios_app_adhoc--ios_app_development--ios_app_inhouse--ios_app_store--mac_app_development--mac_app_direct--mac_app_store--mac_catalyst_app_development--mac_catalyst_app_direct--mac_catalyst_app_store--tvos_app_adhoc--tvos_app_development--tvos_app_inhouse--tvos_app_store), Codemagic will fetch or create the relevant provisioning profile and certificate to code sign the build.

## Provide signing files manually

{{<notebox>}}
If you are using [Codemagic Teams](../teams/teams), then signing files, such as certificates and provisioning profiles, can be managed under the [Code signing identities](./code-signing-identities) section in the team settings and do not have to be uploaded as environment variables as in the below instructions. However, note that some functionality may be limited.
{{</notebox>}}

In order to use manual code signing, save your **signing certificate**, the **certificate password** (if the certificate is password-protected) and the **provisioning profile** in the **Environment variables** section in Codemagic UI. Select **Secure** to encrypt the values. Note that binary files (i.e. provisioning profiles & .p12 certificate) have to be [`base64 encoded`](../variables/environment-variable-groups/#storing-sensitive-valuesfiles) locally before they can be saved to **Environment variables** and decoded during the build.

```yaml
environment:
  groups:
    - app_store_credentials
    - provisioning_profile
    
  # Add the above mentioned group environment variables in Codemagic UI (either in Application/Team variables):
    # APP_CERTIFICATE
    # APP_CERTIFICATE_PASSWORD
    # INSTALLER_CERTIFICATE
    # INSTALLER_CERTIFICATE_PASSWORD
    # PROVISIONING_PROFILE
```

Then add the code signing configuration and the commands to code sign the build in the scripts section, after all the dependencies are installed, right before the build commands.

```yaml
scripts:
  ... your dependencies installation
  - name: Set up keychain to be used for codesigning using Codemagic CLI 'keychain' command
    script: keychain initialize
  - name: Set up Provisioning profiles from environment variables
    script: |
      PROFILES_HOME="$HOME/Library/MobileDevice/Provisioning Profiles"
      mkdir -p "$PROFILES_HOME"
      PROFILE_PATH="$(mktemp "$PROFILES_HOME"/$(uuidgen).mobileprovision)"
      echo ${PROVISIONING_PROFILE} | base64 --decode > "$PROFILE_PATH"
      echo "Saved provisioning profile $PROFILE_PATH"
  - name: Set up signing certificates
    script: |
      echo $APP_CERTIFICATE | base64 --decode > /tmp/app_certificate.p12
      if [ -z ${APP_CERTIFICATE_PASSWORD+x} ]; then
        # when using a certificate that is not password-protected
        keychain add-certificates --certificate /tmp/app_certificate.p12
      else
        # when using a password-protected certificate
        keychain add-certificates --certificate /tmp/app_certificate.p12 --certificate-password $APP_CERTIFICATE_PASSWORD
      fi

      echo $INSTALLER_CERTIFICATE | base64 --decode > /tmp/installer_certificate.p12
      if [ -z ${INSTALLER_CERTIFICATE_PASSWORD+x} ]; then
        # when using a certificate that is not password-protected
        keychain add-certificates --certificate /tmp/installer_certificate.p12
      else
        # when using a password-protected certificate
        keychain add-certificates --certificate /tmp/installer_certificate.p12 --certificate-password $INSTALLER_CERTIFICATE_PASSWORD
      fi
  - name: Set up code signing settings on Xcode projects
    script: xcode-project use-profiles
  ... your build commands
```

### Packaging the application

To package your application into an `.pkg` Installer package and sign it with the `Mac Installer Distribution` certificate, use the following script:

```yaml
  - name: Package application
    script: |
      set -x

      APP_NAME=$(find $(pwd) -name "*.app")                                        # Command to find the path to your generated app, may be different
      cd $(dirname "$APP_NAME")
      PACKAGE_NAME=$(basename "$APP_NAME" .app).pkg
      xcrun productbuild --component "$APP_NAME" /Applications/ unsigned.pkg  # Create and unsigned package

      # Find the installer certificate commmon name in keychain
      INSTALLER_CERT_NAME=$(keychain list-certificates \
        | jq '.[]
          | select(.common_name
          | contains("Mac Developer Installer"))
          | .common_name' \
        | xargs)
      xcrun productsign --sign "$INSTALLER_CERT_NAME" unsigned.pkg "$PACKAGE_NAME" # Sign the package
      rm -f unsigned.pkg                                                       # Optionally remove the not needed unsigned package
```

Don't forget to specify the path to your generated package in the [artifacts section](../getting-started/yaml/#artifacts).

### Notarizing macOS applications

Notarization is a process where Apple verifies your application to make sure it has a Developer ID code signature and does not consist of malicious content. Notarizing an app during the Codemagic build process is possible using **altool** as follows:

```
xcrun altool --notarize-app -f <file> --primary-bundle-id <bundle_id>
           {-u <username> [-p <password>] | --apiKey <api_key> --apiIssuer <issuer_id>}
           [--asc-provider <name> | --team-id <id> | --asc-public-id <id>]
```
