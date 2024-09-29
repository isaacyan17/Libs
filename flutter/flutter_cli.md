# Flutter create Cli

```
flutter create --help
```

### --project-name

```
flutter create --project-name myapp

 指定项目的名称。如果未指定，将使用目录名称作为项目名称
```

### --org

```
    flutter create --org com.example
    指定你的Flutter package的组织名，通常是一个反向域名。Android的包名, iOS的Bundle ID

```

### --description
提供项目的描述，这将用于`pubspec.yaml`文件中的`description`字段。


### --ios-language

```
    flutter create --ios-language objc
    或者

    flutter create --i objc

    指定iOS项目使用的编程语言。默认是Swift，可以设置为`objc`来使用Objective-C。
```

### --android-language

```
    flutter create --android-language java
    或者

    flutter create --a java

    指定Android项目使用的编程语言。默认是Kotlin，可以设置为`java`来使用Java。
```


### --platforms

```
    flutter create --platforms android,ios

    指定要创建的平台。可以选择`android`、`ios`、`windows`、`linux`、`macos`、`web`。如果未指定，将创建所有平台。
```


### --template

```
    flutter create --template=app
     flutter create --template=plugin
     flutter create --template=package
    flutter create --template=module

    指定要使用的模板。模板可以是一个有效的Flutter项目，也可以是一个Flutter插件。如果未指定，将使用app默认模板。
```
