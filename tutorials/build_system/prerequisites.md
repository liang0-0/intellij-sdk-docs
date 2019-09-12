---
title: Getting Started with Gradle
---

Adding Gradle build support to an IntelliJ Platform Plugin requires a recent distribution of the Gradle build system and IntelliJ IDEA (Community or Ultimate).

### 1.0 Download and Install IntelliJ IDEA

Download and install either IntelliJ IDEA Ultimate or the IntelliJ IDEA Community Edition.

### 1.1 Ensure that 'Gradle' and 'Plugin DevKit' Plugins are Enabled

You can verify that the plugins are enabled by visiting **Settings \| Plugins**.

![Ensure the Gradle plugin is enabled](img/step0_gradle_enabled.png){:width="858px"}

### 1.2 Create a Plugin Project from Scratch

IntelliJ IDEA supports automatically creating new plugin projects using Gradle, with all the necessary `build.gradle`
setup performed automatically. This can also be used to convert an existing plugin to Gradle, if Gradle is not able to 
convert the existing project - in this case, you need to copy over the sources to the new project.

To do so, create a new project in IntelliJ IDEA by opening **File \| New... \| Project**, and select Gradle from the dialog box.
In the "Additional Libraries and Frameworks" page, check "IntelliJ Platform Plugin".

![Select the Gradle facet in the Project Creation Wizard](img/step1_new_gradle_project.png){:width="800px"}

The Project Creation Wizard will now guide you through the Gradle project creation process. You will need to specify a Group ID, Artifact ID, and Version:

![Specify the Group, Artifact, and Version IDs](img/step2_group_artifact_version.png){:width="800px"}

It’s recommended to select the `Use default gradle wrapper` option, that way IntelliJ IDEA will install everything you need to run Gradle tasks itself.

Finally, specify a JVM Gradle will use, it can be the Project JDK. You also configure this path once the project is created via **Settings \| Build, Execution, Deployment \| Build Tools \| Gradle**.

![Verify the JVM is the correct version](img/step3_gradle_config.png){:width="800px"}

### 1.3 Configuring a Gradle Plugin Project
Support for Gradle-based plugin projects is provided by the IntelliJ Platform `gradle-intellij-plugin`.
See the [Gradle plugin README](https://github.com/JetBrains/gradle-intellij-plugin/blob/master/README.md#gradle) for more information. 
For example, to configure the **Sandbox Home** directory's location include the following in the project's `build.gradle` file:
```groovy
intellij {
  sandboxDirectory = "$project.buildDir/myCustom-sandbox"
}
```
See the [IDE Development Instances](/basics/ide_development_instance.md) 
page for more information about default Sandbox Home directory locations and contents.
 
### 1.4 Add Gradle Support to an Existing Plugin 

To add Gradle support to an existing plugin project, create a `build.gradle` file under the root directory, with at least the following contents:

```groovy
buildscript {
    repositories {
        mavenCentral()
    }
}

plugins {
    id "org.jetbrains.intellij" version "0.4.5"
}

apply plugin: 'idea'
apply plugin: 'org.jetbrains.intellij'
apply plugin: 'java'

intellij {
    version 'IC-2016.3' //IntelliJ IDEA 2016.3 dependency; for a full list of IntelliJ IDEA releases please see https://www.jetbrains.com/intellij-repository/releases
    plugins 'coverage' //Bundled plugin dependencies
    pluginName 'plugin_name_goes_here'
}

group 'org.jetbrains'
version '1.2' // Plugin version
```

Then, with the Gradle executable on your system `PATH`, execute the following commands on your system's command line:

```
gradle cleanIdea idea
```

This will clean any existing IntelliJ IDEA configuration files and generate a new Gradle build configuration recognized by IntelliJ IDEA. Once your project refreshes, you should be able to view the Gradle tool window displayed under **View \| Tool Windows \| Gradle**. This indicates that IntelliJ IDEA recognizes the Gradle facet.

### 1.5 Running a Simple Plugin

Now add a new `HelloAction` class to the Java folder, and `plugin.xml` and `pluginIcon.svg` files in the `META-INF` folder.
For more information about `pluginIcon.svg` files, see the [Plugin Icon](/basics/plugin_structure/plugin_icon_file.md) page.

![Gradle directory structure](img/gradle_directory_structure.png){:width="374px"}

```java
{% include /code_samples/gradle_plugin_demo/src/main/java/HelloAction.java %}
```

```java
{% include /code_samples/gradle_plugin_demo/src/main/resources/META-INF/plugin.xml %}
```

Open the Gradle tool window and search for `runIde` task. If it’s not in the list, please hit `Refresh` button on the top. Double-click on it to run it.

![Gradle Tool Window](img/gradle_tasks_in_tool_window.png){:width="398px"}

Or add a new Gradle Run Configuration, configured like so:

![Gradle Run Configuration](img/gradle_run_config.png){:width="800px"}

Launch the new Gradle Run Configuration. From the Run Window, the following output should be visible.

![Gradle task output](img/launched.png){:width="800px"}

Finally, when the IDE launches, there should be a new menu to the right of the **Help** menu. Your plugin is now configured on Gradle.
