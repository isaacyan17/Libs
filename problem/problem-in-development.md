# 开发问题汇总

## Flutter

* `flutter run` 的时候报错： `... major version 61`

> Flutter 3.10依赖java17的版本；项目中使用Java11.需要在编译时指定编译版本为Java11。

```
// flutter doctor

Flutter (Channel stable, 3.10.0, on macOS 13.2 22D49 darwin-arm64, locale
    zh-Hans-CN)
[!] Android toolchain - develop for Android devices (Android SDK version
    32.1.0-rc1)
    ! Some Android licenses not accepted. To resolve this, run: flutter doctor
      --android-licenses
[✓] Xcode - develop for iOS and macOS (Xcode 14.1)
```

解决办法：

1. 查询本机JDK路径：
`/usr/libexec/java_home -V`
```
Matching Java Virtual Machines (2):
    11.0.15 (x86_64) "Oracle Corporation" - "Java SE 11.0.15" /Library/Java/JavaVirtualMachines/jdk-11.0.15.jdk/Contents/Home
    1.8.0_362 (arm64) "Amazon" - "Amazon Corretto 8" /Users/yjq/Library/Java/JavaVirtualMachines/corretto-1.8.0_362/Contents/Home
/Library/Java/JavaVirtualMachines/jdk-11.0.15.jdk/Contents/Home
```

2. 在`gradle.properties` 中配置路径：

`org.gradle.java.home=/Library/Java/JavaVirtualMachines/jdk-11.0.15.jdk/Contents/Home`

## CLI 打包Flutter iOS IPA

```
flutter build ipa --export-method=enterprise

///可用的命令行参数有 --export-method app-store-connect , --export-method ad-hoc、--export-method development 和 --export-method enterprise。

```

* *如果发现打包出来的版本没有跟随Flutter pubspec.yaml中的配置,需要在ios/Runner/Info.plist中修改`CFBundleVersion`  和 `CFBundleShortVersionString`* : 

```
old: 
<key>CFBundleShortVersionString</key>
	<string>$(MARKETING_VERSION)</string>
	<key>CFBundleVersion</key>
	<string>$(CURRENT_PROJECT_VERSION)</string>

--- 

changed: 


<dict>
  ...
  <key>CFBundleVersion</key>
  <string>$(FLUTTER_BUILD_NUMBER)</string>
  <key>CFBundleShortVersionString</key>
  <string>$(FLUTTER_BUILD_NAME)</string>
  ...
</dict>

```