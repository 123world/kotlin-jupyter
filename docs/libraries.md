# Adding new libraries

Generally, there are two ways of adding new library: 
1. [Creating JSON library descriptor](#Creating-library-descriptor)
2. [Integration using Kotlin API](#Integration-using-Kotlin-API)

## Creating library descriptor

To support new `JVM` library and make it available via `%use` magic command you need to create a library descriptor for it.

Check [libraries](../libraries) directory to see examples of library descriptors.

Library descriptor is a `<libName>.json` file with the following fields:
- `properties`: a dictionary of properties that are used within library descriptor
- `description`: a short library description which is used for generating libraries list in README
- `link`: a link to library homepage. This link will be displayed in `:help` command
- `minKernelVersion`: a minimal version of Kotlin kernel which may be used with this descriptor
- `repositories`: a list of maven or ivy repositories to search for dependencies
- `dependencies`: a list of library dependencies
- `imports`: a list of default imports for library
- `init`: a list of code snippets to be executed when library is included
- `initCell`: a list of code snippets to be executed before execution of any cell
- `shutdown`: a list of code snippets to be executed on kernel shutdown. Any cleanup code goes here
- `renderers`: a mapping from fully qualified names of types to be rendered to the Kotlin expression returning output value.
  Source object is referenced as `$it`
- `resources`: a list of JS/CSS resources. See [this descriptor](../src/test/testData/lib-with-resources.json) for example

*All fields are optional

For the most relevant specification see `org.jetbrains.kotlinx.jupyter.libraries.LibraryDescriptor` class.

Name of the file should have the `<name>.json` format where `<name>` is an argument for '%use' command

Library properties can be used in any parts of library descriptor as `$property`

To register new library descriptor:
1. For private usage - create it anywhere on your computer and reference it using file syntax.
2. Alternative way for private usage - create descriptor in `.jupyter_kotlin/libraries` folder and reference
   it using "default" syntax
3. For sharing with community - commit it to [libraries](../libraries) directory and create pull request.

If you are maintaining some library and want to update your library descriptor, create pull request with your update.
After your request is accepted, new version of your library will be available to all Kotlin Jupyter users
immediately on next kernel startup (no kernel update is needed) - but only if they use `%useLatestDescriptors` magic.
If not, kernel update is needed.

## Integration using Kotlin API

You may also add a Kotlin kernel integration to your library using a
[Gradle plugin](https://plugins.gradle.org/plugin/org.jetbrains.kotlin.jupyter.api).

In the following code snippets `<jupyterApiVersion>` is one of the published versions from the link above.
It is encouraged to use the latest stable version.

First, add the plugin dependency into your buildscript.

For `build.gradle`:
```groovy
plugins {
    id "org.jetbrains.kotlin.jupyter.api" version "<jupyterApiVersion>"
}
```

For `build.gradle.kts`:
```kotlin
plugins {
    kotlin("jupyter.api") version "<jupyterApiVersion>"
}
```

This plugin adds dependencies to api and annotations ("scanner") artifacts to your project. You may turn off
the auto-including of these artifacts by specifying following Gradle properties:
 - `kotlin.jupyter.add.api` to `false`.
 - `kotlin.jupyter.add.scanner` to `false`.

Add these dependencies manually using `kotlinJupyter` extension:
```groovy
kotlinJupyter {
    addApiDependency("<version>")
    addScannerDependency("<version>")
}
```

Finally, implement `org.jetbrains.kotlinx.jupyter.api.libraries.LibraryDefinitionProducer` or
`org.jetbrains.kotlinx.jupyter.api.libraries.LibraryDefinition` and mark implementation with
`JupyterLibrary` annotation:

```kotlin
package org.my.lib
import org.jetbrains.kotlinx.jupyter.api.annotations.JupyterLibrary
import org.jetbrains.kotlinx.jupyter.api.*
import org.jetbrains.kotlinx.jupyter.api.libraries.*

@JupyterLibrary
internal class Integration : JupyterIntegration() {
    
    override fun Builder.onLoaded() {
        render<MyClass> { HTML(it.toHTML()) }
        import("org.my.lib.*")
        import("org.my.lib.io.*")
    }
}
```

For more complicated example see [integration of dataframe library](https://github.com/nikitinas/dataframe/blob/master/src/main/kotlin/org/jetbrains/dataframe/jupyter/Integration.kt).

For a further information see docs for:
 - `org.jetbrains.kotlinx.jupyter.api.libraries.JupyterIntegration`
 - `org.jetbrains.kotlinx.jupyter.api.libraries.LibraryDefinitionProducer`
 - `org.jetbrains.kotlinx.jupyter.api.libraries.LibraryDefinition`

### Avoiding using annotation processor
You may want not to use annotation processing for implementations detection.
Then you may refer your implementations right in your buildscript. Note that
no checking for existence will be performed in this case.

The following example shows how to refer aforementioned `Integration` class in your buildscript.
Obviously, in this case you shouldn't mark it with `JupyterLibrary` annotation.

For `build.gradle`:
```groovy
processJupyterApiResources {
    libraryProducers = ["org.my.lib.Integration"]
}
```

For `build.gradle.kts`:
```kotlin
tasks.processJupyterApiResources {
    libraryProducers = listOf("org.my.lib.Integration")
}
```
