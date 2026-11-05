# Overriding `idea.properties` through an environment variable

**Question this answers:** which environment variable points MPS, or a custom RCP built with the MPS build language, at
a custom `idea.properties` file (to relocate the config, system, plugins, and log directories)? What does the variable
name depend on, and how does it differ between MPS versions?

**Short answer:** for MPS itself the variable is `MPS_PROPERTIES`. For a custom RCP it is `<product>_PROPERTIES`, where
`<product>` is the text of the `product` field of the `BuildMps_Branding` in the RCP's build script, used verbatim (it
is not upper-cased). The launcher turns the variable into the JVM option `-Didea.properties.file=<value>`. The platform
then reads that file first, before the other `idea.properties` locations.

Exception: before MPS 2024.3, the native macOS launcher of an RCP used `upper(<script>)_PROPERTIES`, where `<script>` is
the branding's `script` field. See [Version differences](#version-differences).

Researched versions:

- MPS **2023.2**, **2024.1**, **2024.3** (MPS has no 2024.2), **2025.1**, **2026.1**, and the 2026.2 development branch
  (platform build 263).
- Installed macOS distributions: 2023.2.3, 2024.1.6, 2025.1.5 (build `251.29188.797`), 2026.1.2 (build
  `261.27258.1085`).

## How the variable reaches the platform

```text
<product>_PROPERTIES=/path/to/my.properties
  launcher (shell script, .bat, or native launcher)
    -Didea.properties.file=/path/to/my.properties
      PathManager.loadProperties()
        1. file from idea.properties.file system property
        2. <custom options dir>/idea.properties
        3. ~/idea.properties
        4. <bin dir>/idea.properties
```

**Verified:** `PathManager.loadProperties()` in `platform/util/src/com/intellij/openapi/application/PathManager.java`
(intellij-community) builds the file list in this order. `PathManager.PROPERTIES_FILE` is `"idea.properties.file"`. The
platform does not read `<product>_PROPERTIES` itself. Only the launcher does. So when you start the JVM directly (a
custom run configuration, tests, a headless runner), pass `-Didea.properties.file=...` instead.

## Who reads the variable

### Shell and batch scripts (all researched versions)

The build language's `build.startup` generator writes `mps.sh` / `mps.bat` (or the RCP's equivalents). Generator source:
`plugins/mps-build/languages/build.startup/generator/source_gen/jetbrains/mps/build/startup/generator/template/main/QueriesGenerated.java`.

- **Verified:** with no branding, the templates contain the literal `MPS_PROPERTIES`.
- **Verified:** with a branding, the prefix is `BuildString.getText(branding.product)`, with no case conversion:

  ```java
  String.format("if [ -n \"$%s_PROPERTIES\" ]; then", <branding.product text>)
  String.format("  IDE_PROPERTIES_PROPERTY=\"-Didea.properties.file=$%s_PROPERTIES\"", <branding.product text>)
  // .bat: IF NOT "%<product>_PROPERTIES%" == "" SET IDE_PROPERTIES_PROPERTY="-Didea.properties.file=%<product>_PROPERTIES%"
  ```

- **Verified:** the same `branding.product` text also forms `<product>_JDK` and `<product>_VM_OPTIONS`.
- **Verified:** this logic is identical in 2023.2, 2024.1, 2025.1, 2026.1, and the 2026.2 development branch. 2024.3 was
  not checked directly but lies between identical versions.

The MPS distribution's own generated scripts
(`plugins/mps-build/solutions/mpsBuild/source_gen/jetbrains/mps/ide/build/mps.sh` and `mps.bat`) contain
`MPS_PROPERTIES`.

**Inferred:** because the product text is used verbatim, a product name such as `mbeddr` gives `mbeddr_PROPERTIES`, and
a name with spaces or hyphens gives a name that a POSIX shell cannot use as a variable.

### Native launcher, 2024.3 and later (`product-info.json`)

Since MPS 2024.3 (commit `ea001c7021ef`, MPS-37985 "Generate product-info.json"), the `build.startup` generator also
writes `product-info.json` with:

```json
"envVarBaseName": "<branding.product text>",
```

(`propertyMacro_GetValue_6_2` in the same `QueriesGenerated.java`; the MPS distribution gets `"MPS"`.)

**Verified:** the IntelliJ cross-platform native launcher (`native/XPlatLauncher/src/default.rs` in intellij-community)
computes the name as `env_var_base_name + "_PROPERTIES"` and adds `-Didea.properties.file=<path>`. `env_var_base_name`
is `envVarBaseName` from `product-info.json` (a custom command entry can override it). The same base forms `_JDK` and
`_VM_OPTIONS`. The value is read with `get_path_from_env_var(&env_var_name, false)`; its checks on the path were not
examined.

**Verified:** the macOS binaries `MPS.app/Contents/MacOS/mps` of MPS 2025.1.5 and 2026.1.2 are this launcher (they
contain its log strings, such as "Looking for custom properties environment variable"), and their `product-info.json`
has `"envVarBaseName": "MPS"`.

### Native macOS launcher, 2024.1 and earlier

**Verified:** the `MPS.app/Contents/MacOS/mps` binaries of MPS 2023.2.3 and 2024.1.6 are the older Objective-C launcher.
They contain the strings `_PROPERTIES`, `-Didea.properties.file=%@`, and `uppercaseString`. `CFBundleExecutable` in
`Info.plist` is `mps`.

**Verified:** that launcher's source (`native/MacLauncher/Launcher.m` in intellij-community, removed in commit
`1f68a2b28967`, 2024-04-18) builds the name from the `idea.executable` JVM property in `Info.plist`, upper-cased:

```objc
NSString *getExecutable(void) {
    return getJVMProperty(@"idea.executable");
}

NSString *getOverridePropertiesPath(void) {
    NSString *variable = [[getExecutable() uppercaseString] stringByAppendingString:@"_PROPERTIES"];
    return [[NSProcessInfo processInfo] environment][variable];
}
```

MPS 2024.1.6 sets `idea.executable` to `mps`, which gives `MPS_PROPERTIES`.

For a custom RCP, the `build.startup` generator fills both `CFBundleExecutable` and `idea.executable` in `Info.plist`
from the branding's **`script`** field (`BuildMps_Branding.script`), not from `product`. In MPS 2023.2 and 2024.1,
`propertyMacro_GetValue_1_2` and `propertyMacro_GetValue_1_12` in the `build.startup` `QueriesGenerated.java` return
`BuildString.getText(branding.script)`. Their template values are the two `"mps"` strings of the Info.plist template.

- **Verified:** both queries return the `script` text (and keep the template value `mps` when there is no branding).
- **Inferred:** query 2 is `CFBundleExecutable` and query 12 is `idea.executable`. The template values are registered in
  the same order as the keys in the generated `Info.plist` (`CFBundleExecutable` = `mps`, then `CFBundleIconFile` =
  `mps.icns`; later `idea.vendor.name` = `JetBrains`, then `idea.executable` = `mps`), and query 3 builds
  `<script>.icns`.

So on macOS the variable for an RCP was `upper(<branding.script>)_PROPERTIES`. For MPS's own branding (product `MPS`,
script `mps`) the old and new rules both give `MPS_PROPERTIES`.

### When the macOS launcher changed

MPS had no 2024.2 release: the repository has `2024.1` and `2024.3` release branches and tags, and none for 2024.2.

**Verified:** MPS 2024.3 is the first release with the cross-platform launcher.

- Commit `e565f5a6cf16` (2024-10-30, "Hardcoded product-info for xplat launcher", first tag `2024.3.0`) adds a
  `product-info.json` and explains that the new xplat launcher only understands `product-info.json`.
- Commit `ea001c7021ef` (2024-12-06, MPS-37985, first tag `2024.3.0`) makes the build language generate
  `product-info.json` with `envVarBaseName`.
- In tag `2024.1.6`, `build/mpsDistribution.xml` puts the checked-in `build/resources/mps` binary into
  `MPS.app/Contents/MacOS`. In tag `2024.3.0` it takes `Contents/MacOS` from the downloaded platform macOS archive
  instead. The old Objective-C launcher was removed from intellij-community in April 2024, before platform 243.

## Version differences

| MPS version      | Shell / batch script            | Native macOS launcher                                  |
| ---------------- | ------------------------------- | ------------------------------------------------------ |
| 2023.2, 2024.1   | `<branding.product>_PROPERTIES` | `upper(<branding.script>)_PROPERTIES`                  |
| 2024.3 and later | `<branding.product>_PROPERTIES` | `<branding.product>_PROPERTIES` (via `envVarBaseName`) |

For MPS itself every row gives `MPS_PROPERTIES`. For a custom RCP the macOS name changed in two ways in 2024.3: from
`script` to `product`, and from upper case to as written. Before 2024.3, the `.app` and the shell script could expect
different names. An RCP with product `mbeddr` and script `mbeddr` used `MBEDDR_PROPERTIES` for the `.app` and
`mbeddr_PROPERTIES` for the shell script.

## Unresolved

- How the Windows `.exe` launcher of MPS 2024.1 and earlier names the variable.
- No 2024.3 distribution was run or inspected; the 2024.3 launcher is established from the build scripts and commits.
