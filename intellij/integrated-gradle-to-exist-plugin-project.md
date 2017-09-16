Integrate gradle to exist plugin project (v0.1)
===============================================

Reversion History
-----------------
* 2017.09.16 [Chen Wei][chenwei]
  - create draft
  - update version to 0.1

# Add build.gradle

- build.gradle file content
  ```
  buildscript {
      repositories {
          mavenCentral()
      }
  }

  plugins {
      id "org.jetbrains.intellij" version "0.2.17"
  }

  apply plugin: 'org.jetbrains.intellij'

  intellij {
      intellij.localPath '/Applications/IntelliJ IDEA CE.app'
      pluginName 'Demo Plugin'
  }

  jar {
      baseName 'Demo Plugin'
  }

  group 'com.spreadst.devtools'
  version '1.0'
  ```

  Consult [Grale Intellij Plugin][gradle-intellij-plugin] for more information about the configurations.

- Run gradle task

  `gradle cleanIdea`

- Open gradle tools window

  Try open gradle tools window under **View | Tool Windows | Gradle**.

  > Note:
  >
  > If **Gradle** menu disabled, you can close project then reopen it. IDEA will prompt you to import gradle project. Gradle tools window will be available after you successfully import project.

# Adjust source/resource files if needed

Old source and resource files may not contained in the new imported module. This will cause final jar/zip file contains no class files. In this case you should move the original resource file and source code file to the module's corresponding directory.

Usually the final source directory is `src/main/java`, and the resource directory is `src/main/resources`.

# The last method migrate to gradle build

If you can not migrate to gradle build use aforementioned steps, here is the final method:

0. Keep the build.gradle file
1. Remove all current exist modules
2. Close project
3. Import project directory as a gradle project and config as you wish

# References

- [Gradle Support][gradle-support]
- [Gradle Intellij Plugin][gradle-intellij-plugin]

<!-- links -->
[chenwei]: mailto:weichen2046@gmail.com
[gradle-support]: https://www.jetbrains.org/intellij/sdk/docs/tutorials/build_system/prerequisites.html
[gradle-intellij-plugin]: https://github.com/JetBrains/gradle-intellij-plugin