# Gradle

## 依赖树分析插件
[gradle-taskinfo](https://gitlab.com/barfuin/gradle-taskinfo#gradle-taskinfo)

Gradle 4.x版本引入方法：
```groovy
// project/build.gradle
buildscript {
  repositories {
    maven {
      url = uri("https://plugins.gradle.org/m2/")
    }
  }
  dependencies {
    classpath("org.barfuin.gradle.taskinfo:gradle-taskinfo:2.1.0")
  }
}

```

```groovy
//in app/build.gradle
apply plugin: 'org.barfuin.gradle.taskinfo'

```

Gradle 7.x版本：

```groovy
plugins {
    id 'org.barfuin.gradle.taskinfo' version '2.1.0'
}

```

示例： 打印任务的依赖树

```groovy
./gradlew tiTree assemble
```

```
:assemble                             (org.gradle.api.DefaultTask)
+--- :jar                             (org.gradle.api.tasks.bundling.Jar)
|    `--- :classes                    (org.gradle.api.DefaultTask)
|         +--- :compileJava           (org.gradle.api.tasks.compile.JavaCompile)
|         `--- :processResources      (org.gradle.language.jvm.tasks.ProcessResources)
+--- :javadocJar                      (org.gradle.api.tasks.bundling.Jar)
|    `--- :javadoc                    (org.gradle.api.tasks.javadoc.Javadoc)
|         `--- :classes               (org.gradle.api.DefaultTask)
|              +--- :compileJava      (org.gradle.api.tasks.compile.JavaCompile)
|              `--- :processResources (org.gradle.language.jvm.tasks.ProcessResources)
`--- :sourcesJar                      (org.gradle.api.tasks.bundling.Jar)

```

