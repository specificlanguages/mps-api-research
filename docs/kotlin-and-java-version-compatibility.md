# Kotlin and Java compatibility across MPS versions

MPS bundles a Kotlin compiler, Kotlin standard library, and Kotlin metadata reader. Libraries and plugins loaded into
MPS therefore have three independent compatibility boundaries:

- the Kotlin metadata version understood by the MPS release, and
- the Kotlin standard-library API available at runtime, and
- the Java APIs and class-file version understood by the MPS runtime JDK.

This note records the Kotlin and Java versions in recent MPS releases and explains how to compile a single JVM library
for more than one MPS line. It does not claim that every artifact produced by the listed compilers works with every
other compiler or runtime version; that depends on the configured Kotlin language, Kotlin API, and JVM target versions.

## Bundled versions

| MPS distribution | Bundled Kotlin | Evidence in the distribution                                               |
| ---------------- | -------------- | -------------------------------------------------------------------------- |
| 2023.2.2         | 1.9.0          | `plugins/mps-kotlin/lib/kotlin-stdlib-1.9.0.jar`                           |
| 2024.1.2         | 1.9.20         | `plugins/mps-kotlin/lib/kotlin-stdlib-1.9.20.jar`                          |
| 2024.3.1         | 1.9.20         | `plugins/mps-kotlin/lib/kotlin-stdlib-1.9.20.jar`                          |
| 2025.1.1         | 2.1.0          | `plugins/mps-kotlin/lib/kotlin-stdlib-2.1.0.jar`                           |
| 2026.1           | 2.3.0          | `plugins/mps-kotlin/lib/kotlin-stdlib-2.3.0.jar`                           |
| 262.9437.325     | 2.3.0          | `plugins/mps-kotlin/lib/kotlin-stdlib-2.3.0.jar` in the prerelease archive |

Each distribution also contains its compiler under `plugins/mps-kotlin/kotlinc/`. The unversioned
`kotlinc/lib/kotlin-stdlib.jar` has the same size and contents as the versioned library shown above; compiler jars do
not encode the version in their filename.

The corresponding `kotlin.version` build properties were verified in these JetBrains/MPS revisions:

- 2023.2: [`d417ca5e`](https://github.com/JetBrains/MPS/tree/d417ca5e8c3358461fd617000f79cc70eff07a5b),
  `build/version.properties` specifies `kotlin.version=1.9.0`.
- 2024.1: [`6236c407`](https://github.com/JetBrains/MPS/tree/6236c4073eac3cde78506add6b0fa90601d76009),
  `build/version.properties` specifies `kotlin.version=1.9.20`.
- 2025.1: [`f4d90532`](https://github.com/JetBrains/MPS/tree/f4d90532bcac5e0339b3161cec38abf49567cffb),
  `build/version.properties` specifies `kotlin.version=2.1.0`.
- 2026.1: [`499c4a0f`](https://github.com/JetBrains/MPS/tree/499c4a0fe6bfe7f95462c33966c72192158446ca),
  `build/version.properties` specifies `kotlin.version=2.3.0`.
- 262 development line: [`651a60b1`](https://github.com/JetBrains/MPS/tree/651a60b1cf3b890a2ae23f417734d94137962742),
  `build/version.properties` specifies `kotlin.version=2.3.0`.

The 2025.1 distributions additionally contain `lib/kotlin-metadata-jvm-2.1.0.jar`. In 2026.1 and the 262 development
line, MPS ships patched metadata artifacts named `kotlin-metadata-jvm-2.3.0-mps.jar` and
`kotlinx-metadata-jvm-2.3.0-mps.jar`. The latter names are MPS-specific builds, not Maven coordinates for a generally
published Kotlin release.

## Java runtime versions

| MPS distribution | Required Java | Maximum class-file version found | Java release |
| ---------------- | ------------- | -------------------------------- | ------------ |
| 2023.2.2         | 17            | 61                               | 17           |
| 2024.1.2         | 17            | 61                               | 17           |
| 2024.3.1         | 21            | 65                               | 21           |
| 2025.1.1         | 21            | 65                               | 21           |
| 2026.1           | 25            | 69                               | 25           |
| 262.9437.325     | 25            | 69                               | 25           |

The class-file column was established by scanning every class in every JAR under `lib/` and `plugins/` in each generic
distribution. Versioned entries below `META-INF/versions/` in multi-release jars were excluded because they do not by
themselves raise the jar's baseline runtime requirement. Representative classes at the maximum version were:

- MPS 2023.2.2: `com.jetbrains.rd.util.LoggerKt` in `lib/rd.jar`, class-file version 61.
- MPS 2024.1.2: `ai.grazie.DataHolder$Key` in `lib/app.jar`, class-file version 61.
- MPS 2024.3.1 and 2025.1.1: `jetbrains.mps.tool.make.MakeExecutor` in
  `plugins/mps-ant-make/languages/jetbrains.mps.tool.make.jar`, class-file version 65.
- MPS 2026.1: the same `MakeExecutor` class, class-file version 69.
- Build 262.9437.325: `com.intellij.tests.IgnoreException` in `lib/app-backend.jar`, class-file version 69.

The packaged `readme.txt` agrees with Java 17 for MPS 2023.2.2 and 2024.1.2. It is not reliable evidence for later
versions: the MPS 2024.3.1, 2025.1.1, and 2026.1 generic archives all say "JDK 17 or later", although they contain Java
21 or Java 25 class files. JetBrains'
[MPS 2026.1 migration guide](https://www.jetbrains.com/help/mps/migration-guide.html#runtime-upgraded-to-jdk-25)
confirms that MPS 2026.1 runs on JDK 25 and requires class-file version 69 for languages and plugins.

The generic MPS archives do not include a JBR directory even though `product-info.json` refers to `jbr/bin/java` in
versions that contain that descriptor. A headless launcher using the generic archive must select a suitable JDK.
Platform-specific installers include their matching runtime. This distribution difference does not change which JVM
bytecode a library loaded into MPS may use.

### Selecting a JVM target

A library loaded only into stock MPS installations may target the runtime of its oldest supported MPS version. For
example, a library whose compatibility floor is MPS 2023.2.2 may use Java 17 APIs and class-file version 61:

```kotlin
java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

kotlin {
    compilerOptions {
        jvmTarget = JvmTarget.JVM_17
    }
}
```

The JVM target is a consumer requirement, not merely a choice of build JDK. Compiling on JDK 21 while targeting JVM 17
does not permit use of Java 21 APIs when `--release 17` or the equivalent toolchain configuration is enforced.

Code that runs outside the MPS process needs its own compatibility decision. Build-tool plugins and launchers may run in
an older Gradle daemon JVM and start MPS in a separate Java 17, 21, or 25 process. Such tooling can reasonably retain a
lower JVM target even when the code loaded into MPS moves to Java 17.

## What the table does and does not establish

The table identifies the compiler, standard library, and metadata implementation bundled in each MPS distribution. It is
not by itself a complete Kotlin compatibility matrix.

The compiler and build-JDK versions used to build a library do not uniquely determine that library's compatibility.
Kotlin's `languageVersion` controls the language and metadata compatibility level, while `apiVersion` controls which
standard library APIs may be referenced. The JVM target controls class-file compatibility independently of both. A JVM
11 target therefore does not make a library compatible with an older Kotlin compiler or standard library.

Kotlin documents limited forward compatibility for JVM metadata: a compiler can generally consume metadata from the next
language version, but not arbitrary later versions, and compatibility can depend on which features were used. Relying on
this allowance is weaker than compiling the library at the oldest required language level.

No runtime probe was performed for every pair of compiler and MPS versions. In particular, this research does not
establish that an older MPS Kotlin stub loader can read all metadata emitted at a newer language level.

## Compiling a library for multiple MPS lines

For a library that must run in every MPS version in a range, configure all three independent targets explicitly:

```kotlin
import org.jetbrains.kotlin.gradle.dsl.JvmTarget
import org.jetbrains.kotlin.gradle.dsl.KotlinVersion

kotlin {
    compilerOptions {
        jvmTarget = JvmTarget.JVM_17
        languageVersion = KotlinVersion.KOTLIN_1_9
        apiVersion = KotlinVersion.KOTLIN_1_9
    }
}
```

The values above are appropriate for a library loaded only into stock MPS 2023.2.2 and later. They are not requirements
imposed on all MPS libraries. Select the oldest Kotlin and JVM levels actually required by all consumers.

Setting only `apiVersion` is insufficient. It prevents calls to newer standard-library APIs, but a newer
`languageVersion` can still produce metadata that an older compiler or metadata reader cannot consume. Conversely,
setting only `languageVersion` does not prevent accidental use of newer runtime library APIs. Kotlin's
[library compatibility guidelines](https://kotlinlang.org/docs/api-guidelines-backward-compatibility.html#choose-compatible-language-and-api-versions)
recommend considering both settings for published libraries.

The Kotlin Gradle Plugin version may be newer than the configured language and API versions as long as the compiler
still supports those compatibility modes. This enables compiler fixes without raising the consumer compatibility floor.

## Compiler support boundaries for compatibility modes

Kotlin 2.2 removed support for `-language-version=1.6` and `-language-version=1.7`. Kotlin 2.1 reports a deprecation
warning for these modes, while Kotlin 2.2 treats them as errors. A build that intentionally emits Kotlin 1.6-compatible
artifacts must therefore use Kotlin Gradle Plugin 2.1.x or older.

This is a compiler limitation, not an MPS-specific restriction. Raising the language level to 1.8 permits a newer Kotlin
Gradle Plugin, but it also raises the declared compatibility floor of the published library and needs to be validated
against the oldest supported MPS distribution.

See the Kotlin
[2.2 compatibility guide](https://kotlinlang.org/docs/compatibility-guide-22.html#drop-support-in-language-version-for-1-6-and-1-7)
for the removal and its deprecation cycle.

Kotlin 1.9 compatibility mode remains available for Kotlin/JVM through compiler 2.3.x. Kotlin 2.2 reports a deprecation
warning for it, and Kotlin 2.4 removes it. A JVM library intentionally emitting Kotlin 1.9-compatible artifacts must
therefore use Kotlin Gradle Plugin 2.3.x or older. See the Kotlin
[2.4 release notes](https://kotlinlang.org/docs/whatsnew24.html#breaking-changes-and-deprecations) for the removal.

## Upgrade checklist

When upgrading Kotlin for a library loaded into MPS:

1. Keep `languageVersion`, `apiVersion`, and `jvmTarget` explicit.
2. Check that the new compiler still accepts the chosen compatibility modes.
3. Compile and test with a JDK new enough to read the MPS dependencies, even if the published library targets an older
   class-file version.
4. Inspect the published runtime dependencies; a low `apiVersion` does not automatically replace a newer declared
   `kotlin-stdlib` dependency with an older one.
5. Test the produced artifact in the oldest supported MPS distribution, not only the newest one.
6. If the language or API version is raised, test every supported MPS line whose bundled Kotlin version is older than
   the new level.
7. Re-check `build/version.properties`, `plugins/mps-kotlin/lib/`, the metadata jars, and the distribution's class-file
   versions when adding a new MPS line.

## Re-verification triggers

- An MPS release changes `kotlin.version` or replaces its metadata implementation.
- An MPS release changes its runtime JDK or begins shipping classes with a newer class-file version.
- A Kotlin compiler removes another old language/API compatibility mode.
- A library begins using compiler plugins or Kotlin features that can change emitted metadata independently of common
  language features.
- An MPS plugin or standalone product supplies a different Kotlin runtime than the stock MPS distribution.
