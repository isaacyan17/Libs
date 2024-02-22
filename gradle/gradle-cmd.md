# Gradlew 命令
- 查看所有任务
```info
./gradlew tasks --all 
```

- 查看构建版本
```info
 ./gradlew build -v
./gradlew -v
```

- 只编译清单文件，并查看具体日志，快速定位清单文件报错
```info
gradlew :app:processDebugManifest --stacktrace：
```

- 查看项目的依赖都依赖了哪些库
```info
gradlew :app:dependencies 
```

- 对某个module [moduleName] 的某个任务[TaskName] 运行
```info
/gradlew:moduleName:taskName
```

- 编译并安装debug/release包
```info
./gradlew installDebug   
./gradlew installRelease
```

-  强制更新最新依赖，清除构建并构建
```info
./gradlew clean --refresh-dependencies build
```

- 编译并打Release的包
```info
./gradlew assembleRelease
./gradlew aR
```
