# Hello Detox

A sample app for React Native [Detox](https://github.com/wix/Detox) testing, and running them on Codemagic.

## Codemagic YAML template

You can use the following `codemagic.yaml` template to run the Detox test on Codemagic CI/CD. It consists of both Android and iOS workflow.

> Modify the workflow as per your build pipeline. Also, don't forget to add proper environment variables and publishing email before running this workflow on CI/CD.

```yaml
workflows:
  android-workflow:
    name: Android Detox
    environment:
      vars:
          # Android Keystore environment variables
          CM_KEYSTORE: Encrypted(...) # <-- Put your encrypted keystore file here
          CM_KEYSTORE_PASSWORD: Encrypted(...) # <-- Put your encrypted keystore password here
          CM_KEY_ALIAS_PASSWORD: Encrypted(...) # <-- Put your encrypted keystore alias password here
          CM_KEY_ALIAS_USERNAME: Encrypted(...) # <-- Put your encrypted keystore alias username here
      node: latest
    scripts:
      - name: Install npm dependencies
        script: |
          npm install
      - name: Install detox dependencies
        script: |
          brew tap wix/brew
      - name: Install detox-cli
        script: |
          npm install -g detox-cli
          cd $FCI_BUILD_DIR && npm install detox --save-dev
      - name: Set Android SDK location
        script: |
          echo "sdk.dir=$HOME/programs/android-sdk-macosx" > "$FCI_BUILD_DIR/android/local.properties"
      - name: Set up keystore
        script: |
          echo $CM_KEYSTORE | base64 --decode > /tmp/keystore.keystore
          cat >> "$FCI_BUILD_DIR/android/key.properties" <<EOF
          storePassword=$CM_KEYSTORE_PASSWORD
          keyPassword=$CM_KEY_ALIAS_PASSWORD
          keyAlias=$CM_KEY_ALIAS_USERNAME
          storeFile=/tmp/keystore.keystore
          EOF
      - name: Build with Detox
        script: |
          detox build -c android.emu.release
      - name: Test with Detox
        script: |
          detox test -c android.emu.release
      - name: Build Android release
        script: |
          cd android
          ./gradlew assembleRelease
    artifacts:
      - android/app/build/outputs/**/*.apk
    publishing:
        email:
            recipients:
                - user@example.com # enter your email
  ios-workflow:
    name: iOS Detox
    environment:
        vars:
            XCODE_WORKSPACE: "<xcode_workspace_name>.xcworkspace" # <-- Put the name of your Xcode workspace here
            XCODE_SCHEME: "<scheme_name>" # <-- Put the name of your Xcode scheme here
            # For Manual Code Signing
            # More info about code signing here:
            # https://docs.codemagic.io/code-signing-yaml/signing-ios/
            CM_CERTIFICATE: Encrypted(...) # <-- Put your encrypted certificate here
            CM_CERTIFICATE_PASSWORD: Encrypted(...) # <-- Put your encrypted certificate password here
            CM_PROVISIONING_PROFILE: Encrypted(...) # <-- Put your encrypted provisioning profile here
        node: latest
        xcode: 11.7
        cocoapods: default        
    scripts:
        - name: Install npm dependencies
          script: |
            npm install
        - name: Install detox dependencies
          script: |
            brew tap wix/brew
            brew install applesimutils
        - name: Install detox-cli
          script: |
            npm install -g detox-cli
        - name: Install CocoaPods dependencies
          script: |
            cd ios && pod install
        - name: Set up keychain
          script: |
            keychain initialize
        - name: Set up provisioning profiles
          script: |
            PROFILES_HOME="$HOME/Library/MobileDevice/Provisioning Profiles"
            mkdir -p "$PROFILES_HOME"
            PROFILE_PATH="$(mktemp "$PROFILES_HOME"/$(uuidgen).mobileprovision)"
            echo ${CM_PROVISIONING_PROFILE} | base64 --decode > $PROFILE_PATH
            echo "Saved provisioning profile $PROFILE_PATH"
        - name: Set up signing certificate
          script: |
            echo $CM_CERTIFICATE | base64 --decode > /tmp/certificate.p12
            keychain add-certificates --certificate /tmp/certificate.p12 --certificate-password $CM_CERTIFICATE_PASSWORD
        - name: Set up code signing settings on Xcode project
          script: |
            xcode-project use-profiles
        - name: Build with Detox
          script: |
            detox build -c ios.sim.release
        - name: Test with Detox
          script: |
            detox test -c ios.sim.release
        - name: Build ipa for distribution
          script: |
            xcode-project build-ipa --workspace "$FCI_BUILD_DIR/ios/$XCODE_WORKSPACE" --scheme "$XCODE_SCHEME"
    artifacts:
        - build/ios/ipa/*.ipa
        - /tmp/xcodebuild_logs/*.log
        - $HOME/Library/Developer/Xcode/DerivedData/**/Build/**/*.app
        - $HOME/Library/Developer/Xcode/DerivedData/**/Build/**/*.dSYM 
    publishing:
        email:
            recipients:
                - user@example.com # enter your email
```

## License

Copyright (c) 2020 Souvik Biswas

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
