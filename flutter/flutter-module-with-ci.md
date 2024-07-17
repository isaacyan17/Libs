> Edit at 2021-12-02

## Flutter Module 持续集成原生项目

我们使用 Github Action作为CI平台进行项目集成的测试。要达到如下几点：

- push or tag的时候触发Action开始编译打包。
- Android 和iOS的产物分发到不同的仓库

### Android端
我们的项目集成产物是aar 包，这就意味着我们需要编译    `flutter build aar` .

- 首先在仓库目录下创建 .github/workflow/xxx.yml并开始编辑
- 配置工作流程
```yaml
# 工作流的名字
name: Build aar & cocopad module

# 工作流出发的时机，我们这是任意push的时候就触发工作流。
on:
  push:
#      tags:
#       - 'v*' # Push events to matching v*, i.e. v1.0, v20.15.10
  
  pull_request:
    branches: [ main ]

# 工作流要执行的任务
jobs:
# 任务名称
  build:
    runs-on: ubuntu-latest

    # Note that this workflow uses the latest stable version of the Dart SDK.
    # Docker images for other release channels - like dev and beta - are also
    # available. See https://hub.docker.com/r/google/dart/ for the available
    # images.

# 任务步骤
    steps:
    # 拉取项目代码
      - uses: actions/checkout@v2

# 配置java环境
      - name: Setup Java JDK
        uses: actions/setup-java@v1.4.3
        with:
          java-version: "12.x"
          
 # 配置FLutter环境 我们限定flutter version为1.22.x
      - name: Flutter action
#         run: |
#           git clone https://github.com/flutter/flutter.git
#           cd flutter 
        uses: subosito/flutter-action@v1
        with:
          channel: "stable"
          flutter-version: "1.22.x"
      
      - name: Print Java Version
        run: java -version 
      
      - name: Print Flutter version
        run: flutter --version

# 开始下载项目依赖
      - name: Install dependencies
        run: flutter pub get
      
 # pub get会重新创建.android目录。我们将写好的gradle配置文件复制到对应的目录下     
      - name: Reinit gradle in andorid
        run: |
          cp -f configs/buildM.gradle .android/build.gradle
          cp -f configs/buildA.gradle .android/Flutter/build.gradle
 # build一遍       
      - name: aync & build gradle
        run: |
          cd .android
          ./gradlew assembleDebug
        
 # 开始build Android产物      
      - name: build android 
        run: flutter build aar --release
      
 # 将生成的产物发布到对应的仓库/平台，这里是发布到github仓库的release     
      - name: Release apk
        uses: ncipollo/release-action@v1
        with:
          artifacts: "build/host/outputs/repo/com/flutter/simlink/flutter_release/1.0/*.aar"
          tag: "v1.0.6"
          token: "${{secrets.RELEASE_TOKEN}}"
      
      

      # Uncomment this step to verify the use of 'dart format' on each commit.
      # - name: Verify formatting
      #   run: dart format --output=none --set-exit-if-changed .

      # Consider passing '--fatal-infos' for slightly stricter analysis.

      # Your project will need to have tests in test/ and a dependency on
      # package:test for this step to succeed. Note that Flutter projects will
      # want to change this to 'flutter test'.
```

- 需要注意的点：

``flutter pub get` 这个命令会重新生成.android文件目录，我们的设想是最终将所有产物打成一个aar文件。
所以我们在.android/build.gradle中引入了合并打包的方案： `fat-aar android. `这就需要我们在pub get之后依然沿用之前的.android/build.gradle 文件。
所以我们将gradle文件保留起来，执行pub get 之后，使用copy命令写入新生产的.android 目录下。

#### 参考
[github action 发布Android应用](https://segmentfault.com/a/1190000021863232)

[github 实例](https://github.com/OpenIoTHub/OpenIoTHub/actions/runs/300617015)

## 原生项目gitLab-ci 自动化构建

windows 打开管理员权限PowerShell
```bash
在用户组powershell上 输入: 
start-process PowerShell -verb runas
```
