---
title: DUBBO SPI 解析
date: 2019-10-17 23:54:00
updated: 2019-10-17 23:54:00
tags:
  - DUBBO
  - SPI
  - RPC
categories: 
  - 实践
  - 技术小结
comments: true
permalink: tech/dubbo_spi.html    
---

# 0. 背景

最近因为项目需要，需要基于 Dubbo 增加一个 filter，拦截所有的 RPC 请求，打印出请求的相关信息。  
我自定义的 Filter 作为提供者拦截器一共有三步：
1. 写一个自定义类，实现自 Filter 接口
2. 在 META-INF/dubbo 下增加一个 org.apache.dubbo.rpc.Filter 的文件名，内容为自定义 Filter 的全路径
3. 自定义 Filter 类上增加一个 @Activate(group = Constants.PROVIDER, order = -999) 注解  

因此有个疑问，它是如何生效的呢？我在哪个地方让它生效的呢？因此在参考了一些博客和项目后，有了我这篇总结，分析了 Dubbo SPI 的原理以及我还未使用到的 AOP、IOC 特性是如何实现的。  
  
reference:  
1. http://dubbo.apache.org/zh-cn/docs/source_code_guide/dubbo-spi.html
2. http://dubbo.apache.org/zh-cn/blog/introduction-to-dubbo-spi.html
3. http://dubbo.apache.org/zh-cn/blog/introduction-to-dubbo-spi-2.html
4. https://github.com/LiWenGu/iris-java.git（轻量级微内核插件机制的 Java RPC 框架）

<!--more-->

---

如果让你实现类似 SPI 功能，你会如何实现？

# 1. 加载文件反射动态的获得所有扩展相关类

1. 读取名称为接口全路径的文件，文件内容为具体实现类的全路径，将内容全部保存为 Map

```java
/**
     *
     * @param extName2Class 普通的扩展类 Map
     * @param name2Attributes 解析的参数 Map
     * @param name2Wrapper wrapper 包装类 Map
     * @param classLoader
     * @param url
     */
    private void readExtension0(Map<String, Class<?>> extName2Class, Map<String, Map<String, String>> name2Attributes, Map<String, Class<? extends T>> name2Wrapper, ClassLoader classLoader, URL url) {
        BufferedReader reader = null;
        try {
            reader = new BufferedReader(new InputStreamReader(url.openStream(), "utf-8"));
            String line;
            while ((line = reader.readLine()) != null) {
                String config = line;

                // delete comments 如果为注释
                final int ci = config.indexOf('#');
                if (ci >= 0) config = config.substring(0, ci);
                config = config.trim();
                if (config.length() == 0) continue;

                try {
                    String name = null;
                    String body = null;
                    String attribute = null;
                    // 例如内容为 impl1=com.leibangzhu.coco.ext1.impl.SimpleExtImpl1
                    // 通过 = 分割为 name 和 body
                    int i = config.indexOf('=');
                    if (i > 0) {
                        name = config.substring(0, i).trim();
                        body = config.substring(i + 1).trim();
                    }
                    // 没有配置文件中没有扩展点名，从实现类的Extension注解上读取。
                    if (name == null || name.length() == 0) {
                        throw new IllegalStateException(
                                "missing extension name, config value: " + config);
                    }
                    // 例如内容为 impl1=com.leibangzhu.coco.ext4.impl.WithAttributeExtImpl1(k1=v1,k2,k3=v3,k4=,k5=v5)
                    // 通过 ( 分割获取后面的内容为参数，这里为 map
                    int j = config.indexOf("(", i);
                    if (j > 0) {
                        if (config.charAt(config.length() - 1) != ')') {
                            throw new IllegalStateException(
                                    "missing ')' of extension attribute!");
                        }
                        body = config.substring(i + 1, j).trim();
                        attribute = config.substring(j + 1, config.length() - 1);
                    }

                    Class<? extends T> clazz = Class.forName(body, true, classLoader).asSubclass(type);
                    if (!type.isAssignableFrom(clazz)) {
                        throw new IllegalStateException("Error when load extension class(interface: " +
                                type.getName() + ", class line: " + clazz.getName() + "), class "
                                + clazz.getName() + "is not subtype of interface.");
                    }
                    // 例如内容为 *adaptive=com.leibangzhu.coco.ext9.impl.ManualAdaptive
                    // 通过 * 得到该类为动态生成类，但是在此版本还未生效
                    if (name.startsWith(PREFIX_ADAPTIVE_CLASS)) {
                        if (adaptiveClass == null) {
                            adaptiveClass = clazz;
                        } else if (!adaptiveClass.equals(clazz)) {
                            throw new IllegalStateException("More than 1 adaptive class found: "
                                    + adaptiveClass.getClass().getName()
                                    + ", " + clazz.getClass().getName());
                        }
                    } else {
                        // 例如内容为 +wrapper1=com.leibangzhu.coco.ext3.impl.Ext3Wrapper1
                        // 通过 + 得到该类为包装类
                        final boolean isWrapper = name.startsWith(PREFIX_WRAPPER_CLASS);
                        if (isWrapper)
                            name = name.substring(PREFIX_WRAPPER_CLASS.length());

                        String[] nameList = NAME_SEPARATOR.split(name);
                        for (String n : nameList) {
                            if (!isValidExtName(n)) {
                                throw new IllegalStateException("name(" + n +
                                        ") of extension " + type.getName() + "is invalid!");
                            }

                            if (isWrapper) {
                                try {
                                    clazz.getConstructor(type);
                                    name2Wrapper.put(name, clazz);
                                } catch (NoSuchMethodException e) {
                                    throw new IllegalStateException("wrapper class(" + clazz +
                                            ") has NO copy constructor!", e);
                                }
                            } else {
                                try {
                                    clazz.getConstructor();
                                } catch (NoSuchMethodException e) {
                                    throw new IllegalStateException("extension class(" + clazz +
                                            ") has NO default constructor!", e);
                                }
                                if (extName2Class.containsKey(n)) {
                                    if (extName2Class.get(n) != clazz) {
                                        throw new IllegalStateException("Duplicate extension " +
                                                type.getName() + " name " + n +
                                                " on " + clazz.getName() + " and " + clazz.getName());
                                    }
                                } else {
                                    extName2Class.put(n, clazz);
                                }
                                name2Attributes.put(n, parseExtAttribute(attribute));

                                if (!extClass2Name.containsKey(clazz)) {
                                    extClass2Name.put(clazz, n); // 实现类到扩展点名的Map中，记录了一个就可以了
                                }
                            }
                        }
                    }
                } catch (Throwable t) {
                    IllegalStateException e = new IllegalStateException("Failed to load config line(" + line +
                            ") of config file(" + url + ") for extension(" + type.getName() +
                            "), cause: " + t.getMessage(), t);
                    logger.warn("", e);
                    extClassLoadExceptions.put(line, e);
                }
            } // end of while read lines
        } catch (Throwable t) {
            logger.error("Exception when load extension class(interface: " +
                    type.getName() + ", class file: " + url + ") in " + url, t);
        } finally {
            if (reader != null) {
                try {
                    reader.close();
                } catch (Throwable t) {
                    // ignore
                }
            }
        }
    }
```

# 2. wrapper 包装的 IOC 实现

```java

public T getExtension(String name, List<String> wrappers) {
    // 获取需要被包装的扩展类
    T instance = getExtension(name);
    return createWrapper(instance, wrappers);
}

private T createWrapper(T instance, List<String> wrappers) {
    for (String name : wrappers) {
        try {
            // 通过构造方法对 wrapper 进行初始化
            instance = injectExtension(name2Wrapper.get(name).getConstructor(type).newInstance(instance));
        } catch (Throwable e) {
            throw new IllegalStateException("Fail to create wrapper(" + name + ") for extension point " + type);
        }
    }

    return instance;
}
    
private T injectExtension(T instance) {
        try {
            for (Method method : instance.getClass().getMethods()) {
                if (method.getName().startsWith("set")
                        && method.getParameterTypes().length == 1
                        && Modifier.isPublic(method.getModifiers())) {
                    Class<?> pt = method.getParameterTypes()[0];
                    if (pt.isInterface() && withExtensionAnnotation(pt)) {
                        if (pt.equals(type)) { // avoid obvious dead loop TODO avoid complex nested loop setting?
                            logger.warn("Ignore self set(" + method + ") for class(" +
                                    instance.getClass() + ") when inject.");
                            continue;
                        }

                        try {
                            // 获取被包装类的动态类，该动态类注解的 Adaptive 方法都被代理了
                            Object adaptive = getExtensionLoader(pt).getAdaptiveInstance();
                            method.invoke(instance, adaptive);
                        } catch (Exception e) {
                            logger.error("Fail to inject via method " + method.getName()
                                    + " of interface to extension implementation " + instance.getClass() +
                                    " for extension point " + type.getName() + ", cause: " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```

在例子中即可以使用构造方法进行的注入，也支持 set 方法进行注入。但是不支持循环依赖。

# 3. 根据参数在运行时动态切换实现类

当你的方法上有 @Adaptive 注解，会根据你的参数来动态执行不同实现类的方法  
原理在于动态代理该方法，在执行的时候，根据参数获取不同实现类名称，进而使用不同实现类执行：

```java
private T createAdaptiveInstance() throws IllegalAccessException, InstantiationException {
    // 找到有 adaptive 的注解方法
    checkAndCollectAdaptiveInfo0();

    // 有AdaptiveClass（在扩展点配置文件中声明的类）
    if (adaptiveClass != null) {
        return type.cast(adaptiveClass.newInstance());
    }

    Object p = Proxy.newProxyInstance(ExtensionLoader.class.getClassLoader(), new Class[]{type}, new InvocationHandler() {

        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if (method.getDeclaringClass().equals(Object.class)) {
                String methodName = method.getName();
                if (methodName.equals("toString")) {
                    return "Adaptive Instance for " + type.getName();
                }
                if (methodName.equals("hashCode")) {
                    return 1;
                }
                if (methodName.equals("equals")) {
                    return this == args[0];
                }
                throw new UnsupportedOperationException("not support method " + method +
                        " of Adaptive Instance for " + type.getName());
            }

            if (!adaptiveMethod2ArgIndex.containsKey(method)) {
                throw new UnsupportedOperationException("method " + method.getName() +
                        " of interface " + type.getName() + " is not adaptive method!");
            }

            int confArgIdx = adaptiveMethod2ArgIndex.get(method);
            Object arg = args[confArgIdx];
            NameExtractor nameExtractor = adaptiveMethod2Extractor.get(method);
            // 根据参数来获取扩展类名称，例如参数为 {"key", "impl1"}，那么扩展类则为 impl1
            String extName = nameExtractor.extract(arg);
            if (extName == null) extName = defaultExtension;
            if (extName == null)
                throw new IllegalStateException("Fail to get extension(" + type.getName() +
                        ") name from argument(" + arg + ") use keys(" + Arrays.toString(adaptiveMethod2Keys.get(method)) + ")");
            return method.invoke(ExtensionLoader.this.getExtension(extName), args);
        }
    });

    return type.cast(p);
}

public String extract(Object argument) {
    if (argument == null) {
        throw new IllegalArgumentException("adaptive " + parameterType.getName() +
                " argument == null");
    }

    Object sourceObject = argument;
    // 如果参数是一个包装类，则从该类中的的 get 中获取，参考 ConfigHolder 类 @FromAttribute("config")
    if (fromAttribute != null) {
        try {
            sourceObject = fromGetter.invoke(sourceObject);
            if (null == sourceObject) {
                throw new IllegalArgumentException("adaptive " + parameterType.getName() +
                        " argument " + fromGetter.getName() + "() == null");
            }
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Fail to get attribute " +
                    fromGetter.getName() + ", cause: " + e.getMessage(), e);
        } catch (InvocationTargetException e) {
            throw new IllegalStateException("Fail to get attribute " +
                    fromGetter.getName() + ", cause: " + e.getMessage(), e);
        }
    }
    return getFromMap(sourceObject, adaptiveKeys);
}

/**
 *
 * @param obj   {"simple.ext": "impl1"}
 * @param keys  根据 key 获取 obj 参数的值作为真正的实现类
 *              该值优先取 Adaptive 注解的 value 字段，再取类名，例如：NoDefaultExt，则取 no.default.ext 
 * @return
 */
private static String getFromMap(Object obj, String[] keys) {
    @SuppressWarnings("unchecked")
    Map<String, Object> map = (Map<String, Object>) obj;
    // 优先第一个
    for (String key : keys) {
        Object value = map.get(key);
        if (value == null) {
            continue;
        }
        return value.toString();
    }
    return null;
}
```

具体为，当参数为 Map 时，并且请求执行的方法带有 @Adaptive 时，会根据 Map 参数里面的某个 key 的 value，来决定使用

# 4. 完整线路

![][0]

[0]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/somephoto/2019-10-17_DUBBO_SPI%E5%8E%9F%E7%90%86.jpg