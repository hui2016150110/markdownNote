```groovy
/**
 * The first line in the build configuration applies the Android plugin for
 * Gradle to this build and makes the android block available to specify
 * Android-specific build options.
 */

plugins {
  id 'com.android.application'
}

/**
 * The android block is where you configure all your Android-specific
 * build options.
 */

android {

    /**
     * compileSdkVersion specifies the Android API level Gradle should use to
     * compile your app. This means your app can use the API features included in
     * this API level and lower.
     */

    compileSdkVersion 28

    /**
     * buildToolsVersion specifies the version of the SDK build tools, command-line
     * utilities, and compiler that Gradle should use to build your app. You need to
     * download the build tools using the SDK Manager.
     *
     * This property is optional because the plugin uses a recommended version of
     * the build tools by default.
     */

    buildToolsVersion "30.0.2"

    /**
     * The defaultConfig block encapsulates default settings and entries for all
     * build variants, and can override some attributes in main/AndroidManifest.xml
     * dynamically from the build system. You can configure product flavors to override
     * these values for different versions of your app.
     */

    defaultConfig {

        /**
         * applicationId uniquely identifies the package for publishing.
         * However, your source code should still reference the package name
         * defined by the package attribute in the main/AndroidManifest.xml file.
         */

        applicationId 'com.example.myapp'

        // Defines the minimum API level required to run the app.
        minSdkVersion 15

        // Specifies the API level used to test the app.
        targetSdkVersion 28

        // Defines the version number of your app.
        versionCode 1

        // Defines a user-friendly version name for your app.
        versionName "1.0"
    }

    /**
     * The buildTypes block is where you can configure multiple build types.
     * By default, the build system defines two build types: debug and release. The
     * debug build type is not explicitly shown in the default build configuration,
     * but it includes debugging tools and is signed with the debug key. The release
     * build type applies Proguard settings and is not signed by default.
     */

    buildTypes {

        /**
         * By default, Android Studio configures the release build type to enable code
         * shrinking, using minifyEnabled, and specifies the default Proguard rules file.
         */

        release {
              minifyEnabled true // Enables code shrinking for the release build type.
              proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }

    /**
     * The productFlavors block is where you can configure multiple product flavors.
     * This allows you to create different versions of your app that can
     * override the defaultConfig block with their own settings. Product flavors
     * are optional, and the build system does not create them by default.
     *
     * This example creates a free and paid product flavor. Each product flavor
     * then specifies its own application ID, so that they can exist on the Google
     * Play Store, or an Android device, simultaneously.
     *
     * If you declare product flavors, you must also declare flavor dimensions
     * and assign each flavor to a flavor dimension.
     */

    flavorDimensions "tier"
    productFlavors {
        free {
            dimension "tier"
            applicationId 'com.example.myapp.free'
        }

        paid {
            dimension "tier"
            applicationId 'com.example.myapp.paid'
        }
    }
}

/**
 * The dependencies block in the module-level build configuration file
 * specifies dependencies required to build only the module itself.
 * To learn more, go to Add build dependencies.
 */

dependencies {
    implementation project(":lib")
    implementation 'com.android.support:appcompat-v7:28.0.0'
    implementation fileTree(dir: 'libs', include: ['*.jar'])
}
```