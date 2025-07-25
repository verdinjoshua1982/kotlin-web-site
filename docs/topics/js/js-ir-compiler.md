[//]: # (title: Kotlin/JS IR compiler)

The Kotlin/JS IR compiler backend is the main focus of innovation around Kotlin/JS, and paves the way forward for the
technology. 

Rather than directly generating JavaScript code from Kotlin source code, the Kotlin/JS IR compiler backend leverages a
new approach. Kotlin source code is first transformed into a
[Kotlin intermediate representation (IR)](whatsnew14.md#unified-backends-and-extensibility), 
which is subsequently compiled into JavaScript. For Kotlin/JS, this enables aggressive optimizations, and allows improvements
on pain points that were present in the previous compiler, such as generated code size (through [dead code elimination](#dead-code-elimination)),
and JavaScript and TypeScript ecosystem interoperability, to name some examples.

The IR compiler backend is available starting with Kotlin 1.4.0 through the Kotlin Multiplatform Gradle plugin. To enable it in your
project, pass a compiler type to the `js` function in your Gradle build script:

```groovy
kotlin {
    js(IR) { // or: LEGACY, BOTH
        // ...
        binaries.executable() // not applicable to BOTH, see details below
    }
}
```

* `IR` uses the new IR compiler backend for Kotlin/JS.
* `LEGACY` uses the old compiler backend.
* `BOTH` compiles your project with the new IR compiler as well as the default compiler backend. Use this mode for [authoring libraries compatible with both backends](#authoring-libraries-for-the-ir-compiler-with-backwards-compatibility).

> The old compiler backend has been deprecated since Kotlin 1.8.0. Starting with Kotlin 1.9.0, using compiler types `LEGACY` or `BOTH` leads to an error.
>
{style="warning"}

The compiler type can also be set in the `gradle.properties` file, with the key `kotlin.js.compiler=ir`.
This behaviour is overwritten by any settings in the `build.gradle(.kts)`, however.

## Lazy initialization of top-level properties

For better application startup performance, the Kotlin/JS IR compiler initializes top-level properties lazily. This way,
the application loads without initializing all the top-level properties used in its code. It initializes
only the ones needed at startup; other properties receive their values later when the code that uses them actually runs.

```kotlin
val a = run {
    val result = // intensive computations
    println(result)
    result
} // value is computed upon the first usage
```

If for some reason you need to initialize a property eagerly (upon the application start), mark it with the 
[`@EagerInitialization`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.js/-eager-initialization/){nullable="true"} annotation.

## Incremental compilation for development binaries

The JS IR compiler provides the _incremental compilation mode for development binaries_ that speeds up the development process.
In this mode, the compiler caches the results of `compileDevelopmentExecutableKotlinJs` Gradle task on the module level.
It uses the cached compilation results for unchanged source files during subsequent compilations, making them complete faster,
especially with small changes.

Incremental compilation is enabled by default. To disable incremental compilation for development binaries, add the following line to the project's `gradle.properties`
or `local.properties`:

```none
kotlin.incremental.js.ir=false // true by default
```

> The clean build in the incremental compilation mode is usually slower because of the need to create and populate the caches.
>
{style="note"}

## Output mode

You can choose how the JS IR compiler outputs `.js` files in your project:

* **One per module**. By default, the JS compiler outputs separate `.js` files for each module of a project as a
  compilation result.
* **One per project**. You can compile the whole project into a single `.js` file by adding the following line to
  `gradle.properties`:

  ```none
  kotlin.js.ir.output.granularity=whole-program // 'per-module' is the default
  ```
  
* **One per file**. You can set up a more granular output that generates one (or two, if the file contains exported
  declarations) JavaScript file per each Kotlin file. To enable the per-file compilation mode:

  1. Add the `useEsModules()` function to your build file to support ECMAScript modules:

     ```kotlin
     // build.gradle.kts
     kotlin {
         js(IR) {
             useEsModules() // Enables ES2015 modules
             browser()
         }
     }
     ```
  
     Alternatively, you can use the `es2015` [compilation target](js-project-setup.md#support-for-es2015-features)
     to support ES2015 features in your project.
  
  2. Apply the `-Xir-per-file` compiler option or update your `gradle.properties` file with:
  
     ```none
     # gradle.properties
     kotlin.js.ir.output.granularity=per-file // 'per-module' is the default
     ```

## Minification of member names in production

The Kotlin/JS IR compiler uses its internal information about the relationships of your Kotlin classes and functions to apply more efficient minification, shortening the names of functions, properties, and classes. This reduces the size of resulting bundled applications.

This type of minification is automatically applied when you build your Kotlin/JS application in [production](js-project-setup.md#building-executables) mode, and enabled by default. To disable member name minification, use the `-Xir-minimized-member-names` compiler option:

```kotlin
kotlin {
    js(IR) {
        compilations.all {
            compileTaskProvider.configure {
                compilerOptions.freeCompilerArgs.add("-Xir-minimized-member-names=false")
            }
        }
    }
}
```

## Dead code elimination

[Dead code elimination](https://wikipedia.org/wiki/Dead_code_elimination) (DCE) reduces the size of
the resulting JavaScript code by removing unused properties, functions, and classes.

Unused declarations can appear in cases like:

* A function is inlined and never gets called directly (which always happens except for a few cases).
* A module uses a shared library. Without DCE, parts of the library that you don't use are still included in the resulting bundle.
  For example, the Kotlin standard library contains functions for manipulating lists, arrays, char sequences,
  adapters for DOM, and so on. All of this functionality would require about 1.3 MB as a JavaScript file. A simple
  "Hello, world" application only requires console routines, which are only a few kilobytes for the entire file.

In the Kotlin/JS compiler, DCE is handled automatically:

* DCE is disabled in _development_ bundling tasks, which corresponds to the following Gradle tasks:

    * `jsBrowserDevelopmentRun`
    * `jsBrowserDevelopmentWebpack`
    * `jsNodeDevelopmentRun`
    * `compileDevelopmentExecutableKotlinJs`
    * `compileDevelopmentLibraryKotlinJs`
    * Other Gradle tasks including "development" in their name

* DCE is enabled if you build a _production_ bundle, which corresponds to the following Gradle tasks:

    * `jsBrowserProductionRun`
    * `jsBrowserProductionWebpack`
    * `compileProductionExecutableKotlinJs`
    * `compileProductionLibraryKotlinJs`
    * Other Gradle tasks including "production" in their name

With the [`@JsExport`](js-to-kotlin-interop.md#jsexport-annotation) annotation, you can specify the declarations you want
DCE to treat as roots.

## Preview: generation of TypeScript declaration files (d.ts)

> The generation of TypeScript declaration files (`d.ts`) is [Experimental](components-stability.md). It may be dropped or changed at any time.
> Opt-in is required (see the details below), and you should use it only for evaluation purposes. We would appreciate your feedback on it in [YouTrack](https://youtrack.jetbrains.com/issues?q=%23%7BKJS:%20d.ts%20generation%7D).
>
{style="warning"}

The Kotlin/JS IR compiler is capable of generating TypeScript definitions from your Kotlin code. These definitions can be
used by JavaScript tools and IDEs when working on hybrid apps to provide autocompletion, support static analyzers, and
make it easier to include Kotlin code in JavaScript and TypeScript projects.

If your project produces executable files (`binaries.executable()`), the Kotlin/JS IR compiler collects 
any top-level declarations marked with [`@JsExport`](js-to-kotlin-interop.md#jsexport-annotation) and automatically 
generates TypeScript definitions in a `.d.ts` file.

If you want to generate TypeScript definitions, you have to explicitly configure this in your Gradle build file. 
Add `generateTypeScriptDefinitions()` to your `build.gradle.kts` file in the [`js` section](js-project-setup.md#execution-environments). 
For example:

```kotlin
kotlin {
    js {
        binaries.executable()
        browser {
        }
        generateTypeScriptDefinitions()
    }
}
```

The definitions can be found in `build/js/packages/<package_name>/kotlin` alongside the corresponding
un-webpacked JavaScript code.

## Current limitations of the IR compiler

A major change with the new IR compiler backend is the **absence of binary compatibility** with the default backend.
A library created with the new IR compiler uses a [`klib` format](native-libraries.md#library-format) and can't be used 
from the default backend. In the meantime, a library created with the old compiler is a `jar` with `js` files, which 
can't be used from the IR backend.

If you want to use the IR compiler backend for your project, you need to **update all Kotlin dependencies to versions
that support this new backend**. Libraries published by JetBrains for Kotlin 1.4+ targeting Kotlin/JS already contain all
artifacts required for usage with the new IR compiler backend.

**If you are a library author** looking to provide compatibility with the current compiler backend as well as the new IR
compiler backend, additionally check out the [section about authoring libraries for the IR compiler](#authoring-libraries-for-the-ir-compiler-with-backwards-compatibility)
section.

The IR compiler backend also has some discrepancies in comparison to the default backend. When trying out the new backend,
it's good to be mindful of these possible pitfalls.

* Some **libraries that rely on specific characteristics** of the default backend, such as `kotlin-wrappers`, can display some problems. You can follow the investigation and progress [on YouTrack](https://youtrack.jetbrains.com/issue/KT-40525).
* The IR backend **does not make Kotlin declarations available to JavaScript** by default at all. To make Kotlin declarations visible to JavaScript, they **must be** annotated with [`@JsExport`](js-to-kotlin-interop.md#jsexport-annotation).

## Migrating existing projects to the IR compiler

Due to significant differences between the two Kotlin/JS compilers, making your Kotlin/JS code work with the IR compiler
may require some adjustments. Learn how to migrate existing Kotlin/JS projects to the IR compiler in the [Kotlin/JS IR
compiler migration guide](js-ir-migration.md).

## Authoring libraries for the IR compiler with backwards compatibility

If you're a library maintainer who is looking to provide compatibility with the default backend as well as the new IR
compiler backend, a setting for the compiler selection is available that allows you to create artifacts for both backends,
allowing you to keep compatibility for your existing users while providing support for the next generation of Kotlin compiler.
This so-called `both`-mode can be turned on using the `kotlin.js.compiler=both` setting in your `gradle.properties` file,
or can be set as one of the project-specific options inside your `js` block inside the `build.gradle(.kts)` file:

```groovy
kotlin {
    js(BOTH) {
        // ...
    }
}
```

When in `both` mode, the IR compiler backend and default compiler backend are both used when building a library from your
sources (hence the name). This means that both `klib` files with Kotlin IR as well as `jar` files for the default compiler
will be generated. When published under the same Maven coordinate, Gradle will automatically choose the right artifact
depending on the use case – `js` for the old compiler, `klib` for the new one. This enables you to compile and publish
your library for projects that are using either of the two compiler backends.
