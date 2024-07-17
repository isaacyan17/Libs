# 实现一个JNI工程

> Edit at 2023-08-1

## 创建Jni工程

File->new Project->native c++

## 实现本地方法
### 动态注册
[link](https://juejin.cn/post/6844904192780271630#heading-37)

每一级文件目录下（每一个文件目录需要有一个CmakeLists配置文件），
.c/.cpp文件内重写 JNI_OnLoad()函数.

```
cpp:


jint JNI_OnLoad(JavaVM* vm, void* reserved){
    JNIEnv* env = NULL;
    // 1. 获取 JNIEnv，这个地方要注意第一个参数是个二级指针
    int result = vm->GetEnv(reinterpret_cast<void **>(&env), JNI_VERSION_1_6);
    // 2. 是否获取成功
    if(result != JNI_OK){
        ALOG("获取 env 失败");
        return JNI_VERSION_1_6;
    }
    // 3. 注册方法
    //HOOK_JNI_CLASS_NAME 为Java调用jni方法的路径，例如："com/jni/senthinkprivate/Utils/HookUtils"
    jclass classMainActivity = (env)->FindClass(HOOK_JNI_CLASS_NAME);

    //关联Java方法与native方法
    JNINativeMethod m[] = {
            {"hookString", "()V", (void*)test},
//            {"nativeHook", "(I)I", (void*)hacker_hook},
    };

    // sizeof(methods)/ sizeof(JNINativeMethod)
    result = env->RegisterNatives(classMainActivity,m, sizeof(m) / sizeof(m[0]));

    if(result != JNI_OK){
        ALOG("注册方法失败");
        return JNI_VERSION_1_2;
    }

    //JNI_OnLoad需要一个int的返回值。
    return JNI_VERSION_1_6;
}

```

```
C

JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *vm, void *reserved)
{
    (void)reserved;

    if(NULL == vm) return JNI_ERR;
    ////保存vm
    kJvm = vm;

    /// 动态注册Native方法
    JNIEnv *env;
    if(JNI_OK != (*vm)->GetEnv(vm, (void **)&env, HACKER_JNI_VERSION)) return JNI_ERR;
    if(NULL == env || NULL == *env) return JNI_ERR;

    jclass cls;
    if(NULL == (cls = (*env)->FindClass(env, HOOK_JNI_CLASS_NAME))) return JNI_ERR;

    JNINativeMethod m[] = {
            {"nativeHook", "(I)I", (void *)hacker_hook},
            {"nativeUnhook", "()I", (void *)hacker_unhook},
            {"nativeThreadHook", "()V", (void *)enableThreadHook},
    };
    if(0 != (*env)->RegisterNatives(env, cls, m, sizeof(m) / sizeof(m[0]))) return JNI_ERR;

    return HACKER_JNI_VERSION;
}
```

### 静态注册

方法名关联Java类路径。如Java_com_jni_senthinkprivate_MainActivity_stringFromJNI

## 打包SO库

/app/build.gradle下选择需要编译的平台架构

```
 externalNativeBuild {
            cmake {
                cppFlags ''
                abiFilters "armeabi-v7a","arm64-v8a","x86"
            }
        }
```

## 项目依赖

### cpp目录下多个文件夹编译

- 工程目录如：

```
--- cpp
  --	hook
  	--CMakeLists.txt
  	-- hook.cpp
  -- CMakeLists.txt
  -- test.cpp
```

-  首先在各文件目录下新增CMakeLists.txt配置文件：

```
cmake_minimum_required(VERSION 3.10.2)
set(UTILS_SOURCE
        threadLinker.cpp

        )
add_library(hook
        SHARED
        ${UTILS_SOURCE}

        )
target_link_libraries(hook)
```

- 在/cpp目录下的CMakeLists.txt中加入subDirectory，设置 target 需要链接的库：

```
add_subdirectory(hook)
include_directories(hook)
...
target_link_libraries( # Specifies the target library.
                       native-lib
                       hook

                       # Links the target library to the log library
                       # included in the NDK.
                       ${log-lib} )
```

- Cmake有关释义详见[链接](https://blog.csdn.net/afei__/article/details/81201039)


