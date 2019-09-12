---
title: Library
---

A library is an archive of compiled code (such as JAR files) that your modules depend on.
The IntelliJ Platform supports three types of libraries:

* **Module Library**: the library classes are visible only in this module and the library information is recorded in the module *.iml file.
* **Project Library**: the library classes are visible within the project and the library information is recorded in the project *.ipr file or in .idea/libraries.
* **Global Library**: the library information is recorded in the applicationLibraries.xml file into the `<User Home>/.IntelliJIdea/config/options` directory. Global libraries are similar to project libraries, but are visible for the different projects.

For more information about libraries, refer to
[Library](https://www.jetbrains.com/help/idea/working-with-libraries.html).

## Accessing Libraries and Jars

Package
[libraries](upsource:///platform/projectModel-api/src/com/intellij/openapi/roots/libraries)
provides functionality for working with project libraries and jars.

### How do I get a list of libraries a module depends on?

To get the list of libraries that a module depends on, use `OrderEnumerator.forEachLibrary` method. 
The following snippet demonstrates how you can do this:

```java
final List<String> libraryNames = new ArrayList<String>();
ModuleRootManager.getInstance(module).orderEntries().forEachLibrary(library -> {
  libraryNames.add(library.getName());
  return true;
});
Messages.showInfoMessage(StringUtil.join(libraryNames, "\n"), "Libraries in Module");
```

This sample code outputs a list of libraries that the `module` module depends on.

### How do I get a list of all libraries?

To manage the lists of application and project libraries, the [LibraryTable](upsource:///platform/projectModel-api/src/com/intellij/openapi/roots/libraries/LibraryTable.java) 
class is used. The list of application-level library tables is accessed by calling `LibraryTablesRegistrar.getInstance().getLibraryTable()`,
whereas the list of project-level library tables is accessed through `LibraryTablesRegistrar.getInstance().getLibraryTable(Project)`.
Once you have a `LibraryTable`, you can get the libraries in it by calling `LibraryTable.getLibraries()`.

To get the list of all module libraries defined in a given module, use the following API:
```java
OrderEntryUtil.getModuleLibraries(ModuleRootManager.getInstance(module));
```

### How do I get the library content?

The `Library` class provides the `getUrls` method you can use to get a list of source roots and classes the library includes. To clarify, consider the following code snippet:

```java
StringBuilder roots = new StringBuilder("The " + lib.getName() + " library includes:\n");
roots.append("Sources:\n");
for (String each : lib.getUrls(OrderRootType.SOURCES)) {
  roots.append(each).append("\n");
}
roots.append("Classes:\n");
for (String each : lib.getUrls(OrderRootType.CLASSES)) {
  strRoots.append(each).append("\n");
}
Messages.showInfoMessage(roots.toString(), "Library Info");
```

In this sample code, `lib` is of the [Library](upsource:///platform/projectModel-api/src/com/intellij/openapi/roots/libraries/Library.java) type.

### How do I create a library?

To create a library, you need to perform the following steps:

  * Get a [write action](../../basics/architectural_overview/general_threading_rules.md#readwrite-lock)
  * Obtain the library table to which you want to add the library. Use one of the following, depending on the library level:
      * `LibraryTablesRegistrar.getInstance().getLibraryTable()`
      * `LibraryTablesRegistrar.getInstance().getLibraryTable(Project)`
      * `ModuleRootManager.getInstance(module).getModifiableModel().getModuleLibraryTable()`
  * Create the library by calling `LibraryTable.createLibrary()`
  * Add contents to the library (see below)
  * For a module-level library, commit the modifiable model returned by `ModuleRootManager.getInstance(module).getModifiableModel()`.
  
For module-level libraries, you can also use simplified APIs in the [ModuleRootModificationUtil](upsource:///platform/projectModel-api/src/com/intellij/openapi/roots/ModuleRootModificationUtil.java)
class to add a library with a single API call. You can find an example of using these APIs in the [sample plugin](https://github.com/JetBrains/intellij-sdk-docs/blob/master/code_samples/project_model/src/com/intellij/tutorials/project/model/ModificationAction.java).

### How do I add contents to a library or modify it?

To add or change the roots of a library, you need to perform the following steps:

  * Get a [write action](../../basics/architectural_overview/general_threading_rules.md#readwrite-lock)
  * Get a **modifiable model** for the library, using `Library.getModifiableModel()`
  * Use methods such as `Library.ModifiableModel.addRoot()` to perform the necessary changes
  * Commit the model using `Library.ModifiableModel.commit()`.
  
### How do I add a library dependency to a module?

Use `ModuleRootModificationUtil.addDependency(module, library)` from under a write action.  

### Checking Belonging to a Library

The [ProjectFileIndex](upsource:///platform/projectModel-api/src/com/intellij/openapi/roots/ProjectFileIndex.java) interface implements a number of methods you can use to check whether the specified file belongs to the project library classes or library sources.
You can use the following methods:

* To check if a specified virtual file is a compiled class file use

  ```java
  ProjectFileIndex.isLibraryClassFile(virtualFile)
  ```
* To check if a specified virtual file or directory belongs to library classes use

  ```java
  ProjectFileIndex.isInLibraryClasses(virtualFileorDirectory)
  ```
* To check if the specified virtual file or directory belongs to library sources use

  ```java
  ProjectFileIndex.isInLibrarySource(virtualFileorDirectory)
  ```

See the following [code sample](https://github.com/JetBrains/intellij-sdk-docs/blob/master/code_samples/project_model/src/com/intellij/tutorials/project/model/ProjectFileIndexSampleAction.java)
to see how the method mentioned above can be applied.


More details on libraries can be found in this
[code sample](https://github.com/JetBrains/intellij-sdk-docs/blob/master/code_samples/project_model/src/com/intellij/tutorials/project/model/LibrariesAction.java)
