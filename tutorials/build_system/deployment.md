---
title: Publishing Plugins with Gradle
---

Once you have configured Gradle support, you can automatically build and deploy your plugin to the [JetBrains Plugin Repository](https://plugins.jetbrains.com). To do so, you 
will need to have already published the plugin to the plugin repository. For detailed information, please see the guide to [publishing a plugin](../../basics/getting_started/publishing_plugin.md).

### 2.0 Add your account credentials

In order to deploy a plugin to the plugin repository, you will first need to supply your JetBrains Account credentials. 
We will describe three options to do so: using only the Gradle properties, using environment variables and using arguments to the Gradle task.

#### Using the Gradle properties file
You can store the credentials in the [Gradle properties](https://docs.gradle.org/current/userguide/build_environment.html#sec:gradle_configuration_properties). It is crucial that you do not check these credentials into source control.

To do this, place the following information inside a file called `gradle.properties` under your project's root directory, or inside `GRADLE_HOME/gradle.properties`.

```
intellijPublishUsername=YOUR_USERNAME_HERE
intellijPublishPassword=YOUR_PASSWORD_HERE
```

Then refer to these values in `publishPlugin` task in your `build.gradle` file:

```groovy
publishPlugin {
    username intellijPublishUsername
    password intellijPublishPassword
}
```

If you place a `gradle.properties` file in your project's root directory, please ensure that this file is ignored by your version control tool. For example in Git, you can add the following line to your `.gitignore` file:

```
gradle.properties
```

#### Using environment variables

Alternatively, and possibly slightly more safe because you cannot accidentally commit your credentials to git, you can provide your credentials via the environment variables `ORG_GRADLE_PROJECT_intellijPublishUsername` and `ORG_GRADLE_PROJECT_intellijPublishPassword`.

You can for example do this by providing them in the run configuration with which you run the `publishPlugin` task locally.
To do so, create a Gradle run configuration (if not already done), choose your Gradle project, specify the `publishPlugin` task and then add the mentioned environment variables.

Note that you will still need to put some default values (can be empty) in the Gradle properties, because otherwise you will get a compilation error.

#### Providing parameters to the Gradle task

Similar to using environment variables you can also pass your credentials as parameters to the Gradle task.
You need to provide the parameters `-Dorg.gradle.project.intellijPublishUsername=myusername -Dorg.gradle.project.intellijPublishPassword=mypassword` for example via the command line or by putting this in the arguments of your run configuration.

Note that also in this case you still need to put some default values in your Gradle properties.

### 2.1 Configure your plugin

The gradle-intellij-plugin provides a number of [configuration options](https://github.com/JetBrains/gradle-intellij-plugin#configuration) for customizing how Gradle builds your plugin. One of the most important is the `version`. By default, if you modify the `version` in your build script, the Gradle plugin will automatically update the `<version>` in your `plugin.xml` file. 
 
 The Gradle plugin will also update the `<idea-version since-build=.../>` values within the `plugin.xml` file to match the `intellij.version`, valid until the last release in the current major version, however you can disable this feature by setting the `intellij.updateSinceUntilBuild` option to `false`.

```groovy
plugins {
    // Make sure to check for the latest version at https://plugins.gradle.org/plugin/org.jetbrains.intellij
    // You can also subscribe to releases at https://github.com/JetBrains/gradle-intellij-plugin/releases
    id 'org.jetbrains.intellij' version '0.4.5'
}

intellij {
    version '2018.3'
    pluginName 'idear'
    intellij.updateSinceUntilBuild false //Disables updating since-build attribute in plugin.xml
}

group 'com.jetbrains'
version '1.2' // Update me!
```

When you run `gradle runIde` with a build script containing the above snippet, Gradle will download the appropriate version of IntelliJ IDEA from either a [Snapshot](https://www.jetbrains.com/intellij-repository/snapshots) (time-based) or [Release](https://www.jetbrains.com/intellij-repository/releases) (version based) repository, configure the plugin sandbox, install your plugin, and launch a new instance of the IDE. This task can be run directly from the command line, without any prior tooling assistance. 

### 2.3 Deploy your plugin

The first step when deploying a plugin is to confirm that it works correctly. You may wish to verify this by [installing your plugin from disk](https://www.jetbrains.com/help/idea/managing-plugins.html) on a fresh instance of your target IDE(s). Once you are confident the plugin works as intended, make sure the plugin version is updated, as the JetBrains Plugin repository will not accept multiple artifacts with the same version. To deploy a new version of your plugin to the JetBrains plugin repository, execute the following Gradle command:

```bash
gradle publishPlugin
```

Now check that the most recent version of your plugin appears on the [Plugin Repository](https://plugins.jetbrains.com/). If successfully deployed, any users who currently have your plugin installed on an eligible version of the IntelliJ Platform will be notified of a new update available on the following restart.

You may also deploy plugins to a release channel of your choosing, by configuring the `publishPlugin.channels` property. For example:

```groovy
publishPlugin {
    channels 'beta'
}
```

When empty, this will use the default plugin repository, available to all [JetBrains plugin repository](https://plugins.jetbrains.com/) users. However, you can publish to an arbitrarily-named channel. These non-default release channels are treated as separate repositories for all intents and purposes. When using a non-default release channel, users will need to add a new [custom plugin repository](https://www.jetbrains.com/help/idea/managing-plugins.html#repos) to install your plugin. For example, if you specify `publishPlugin.channels 'canary'`, then users will need to add the `https://plugins.jetbrains.com/plugins/canary/list` repository to install the plugin and receive updates.  Popular channel names include:

* `alpha`: https://plugins.jetbrains.com/plugins/alpha/list
* `beta`: https://plugins.jetbrains.com/plugins/beta/list
* `eap`: https://plugins.jetbrains.com/plugins/eap/list

More information about the available configuration options is at the [documentation of the intellij gradle plugin](https://github.com/JetBrains/gradle-intellij-plugin/blob/master/README.md#publishing-dsl).
