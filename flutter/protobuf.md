# Protobuf in Flutter

## Protobuf
一种轻量的结构化数据结构存储格式，平台无关，可序列化。

```
//指定proto3版本，默认版本时proto2
syntax = "proto3";
//生成类的包名.注意：会在指定路径下按照该包名的定义来生成文件夹
option java_package = "com.jet.protobuf";
// message: 结构体关键字，后面跟类型名（结构体名）
message AdvertisementData{
//string： 类型 ； 
// 等号后面数字表示标识符，每个字段都有唯一的标识符。
  string local_name = 1;
  Int32Value tx_power_level = 2;
  bool connectable = 3;
}
```

## Flutter 项目引入protobuf

### 安装protobuf，protoc_plugin

1. 安装protobuf： brew install protobuf

2. 安装protoc_plugin ： 如果本地单独安装了dart sdk，执行pub global activate protoc_plugin 。如果是flutter sdk ，执行flutter pub global activate protoc_plugin

3. 将protoc_plugin path加入环境配置： 以flutter pub global activate protoc_plugin为例，在.bash_profile中加入： export PATH="$PATH":"$HOME/[your flutter sdk directory path ]/flutter/.pub-cache/bin"

### dart侧编写.proto文件并生成.dart

- 编写.proto文件

```
syntax = "proto3";

option java_package = "com.xx.xx";
option java_outer_classname = "Protos";
option objc_class_prefix = "Protos";

// Wrapper message for `int32`.
//
// Allows for nullability of fields in messages
message Int32Value {
  // The int32 value.
  int32 value = 1;
}

//蓝牙状态
message BluetoothState {
  enum State {
    UNKNOWN = 0;
    UNAVAILABLE = 1;
    UNAUTHORIZED = 2;
    TURNING_ON = 3;
    ON = 4;
    TURNING_OFF = 5;
    OFF = 6;
  };
  State state = 1;
}
```

- 生成.dart 文件

```
以flutter pub global activate protoc_plugin为例。

在.proto文件目录下，执行
protoc --dart_out=../gen --plugin=protoc-gen-dart=$HOME/Documents/workplatform/flutter/.pub-cache/bin/protoc-gen-dart ./flutterble.proto

其中，'../gen' 为生成.dart文件目录路径。
--plugin=...  为指定protoc-gen-dart执行路径

```