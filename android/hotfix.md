# 热修碎片

> 从修复内容区分,可以分为代码修复，资源修复，so库修复


![](https://cdn.nlark.com/yuque/0/2021/jpeg/1227097/1638866242460-b8e4f573-68a3-4a7e-b8ed-c75113ce7e32.jpeg)


## 代码修复
### 热部署
思路：替换`ArtMethod`方法，将fix文件的`ArtMethod`中的成员变量复制到old文件中。=> Native Hook 
难点: 

- 计算一个ArtMethod的内存大小
- 不替换完整的ArtMethod结构体, 即不将ArtMethod成员变量一一对应替换
- 访问权限的问题

解决方案:

- 根据ArtMethod Array 线性排列的特性,构建2个方法, 通过相邻两个方法的起始地址的差值,可以判断出一个ArtMethod的大小.
- 计算单独的ArtMethod以后, 通过memcpy()函数直接将补丁ArtMethod复制过去
- 保证ClassLoader与原来的ClassLoader一致


### 冷启动

思路:  通过ClassLoader查找DexElements数组中的dex,将补丁Dex插入到DexElements的最前面,根据双亲委派的逻辑,DexElements最前面的dex文件会被最先加载,同时也保证了唯一性

## 资源修复

思路: 创建AssetManager, 反射获取addAssetPath, 然后替换原来的AssetManager
## so库修复

思路: so库加载的两种方式:
```java
System.loadLibrary(String libName)：用来加载已经安装的 apk 中的 so
System.load(String pathName)：可以加载自定义路径下的 so
```
如果有补丁 so 下发，我们就调用 System.load 去加载，如果没有补丁 so 没有下发，那么还是调用 System.loadLibrary 去加载系统目录下的 so
