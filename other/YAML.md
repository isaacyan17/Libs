# 基本用法
> -  abc: def 冒号后空格
> - 大小写敏感

## 对象
键值对集合
```yaml
# 对象
person:
  name: abc

# or
person: [name: abc]

```

## 数组
```yaml
ads: 
 - abc
 - def

# or
ads: [abc,def]
```

## 常量
```yaml
msg: 'abc \n def' # 单引号忽略转义字符

# 
msg: "abc \n def" # 双引号识别转义字符
```

## 参数引用
```yaml
name: abc

person:
 name: ${name}
```

## 读取yml/yaml配置内容

```yaml
# .yml/.yaml
name: abc

person: 
 ads: def

ads: 
 - a
 - b

# java
@Value("${name}"）
private String name;

@Value("${person.ads}"）
private String ads;

@Value("${ads[0]"）
private String ads;


# or 使用 Environment读取
@Autowired
private Environment env;

# or 使用@ConfigurationProperties注解

```
