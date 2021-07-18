---
title: dubboSPI的实现
copyright: true
date: 2020-05-20 00:17:24
tags:
	- dubbo SPI
categories:
	- dubbo
---

# dubbo的spi概述
采用spi是为了更好的达到OCP原则（对扩展开放，对修改封闭）。dubbo采用微内核+插件的架构。内核部分功能稳定，面向功能的可拓展性实现都是由插件来完成的，内核只是管理插件和应用插件实现。这样更灵活。

dubbo就是采用spi来加载插件的。

# SPI原理

## jdk中的spi
### 使用
需要在resource目录下的META-INFO/services下新建对应SPI接口名称为名字的文件，然后将实现类的全限类名作为文件内容。

![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/mac_picgo/20210710162401.png)

其文件内容：
![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/mac_picgo/20210710162535.png)

然后利用ServiceLoader接口去加载和使用对应的spi接口即可。
```
public class 测试jdk_spi {

    @Test
    public void testJdkSPI() {
        ServiceLoader<IShot> serviceLoader = ServiceLoader.load(IShot.class);
        Iterator<IShot> iterator = serviceLoader.iterator();
        while (iterator.hasNext()) {
            IShot next = iterator.next();
            System.out.println(next.shot());
        }
    }
}
```

输出：
```
i am a dog
i am a cat
```
### 原理

ServiceLoader接口在调用load时，会创建一个ServiceLoader对应的实例，其中维护了一个providers变量，是一个LinkedHashMap，其会将spi文件中的每个接口实现的名称作为key，具体实例化的实现作为value存储，并且会生成一个LazyIterator作为迭代器的实现。


在调用ServiceLoader迭代器的hasNext和next方法时，会调用到上边的Lazy迭代器，其就是去读取配置文件中的内容，保存到providers中。

```
 private S nextService() {
            if (!hasNextService())
                throw new NoSuchElementException();
            String cn = nextName;
            nextName = null;
            Class<?> c = null;
            try {
                c = Class.forName(cn, false, loader);
            } catch (ClassNotFoundException x) {
                fail(service,
                     "Provider " + cn + " not found");
            }
            if (!service.isAssignableFrom(c)) {
                fail(service,
                     "Provider " + cn  + " not a subtype");
            }
            try {
                S p = service.cast(c.newInstance());
                providers.put(cn, p);
                return p;
            } catch (Throwable x) {
                fail(service,
                     "Provider " + cn + " could not be instantiated",
                     x);
            }
            throw new Error();          // This cannot happen
        }
```

## dubbo中的SPI
dubbo没有直接使用java的SPI来实现自己的插件加载，而是在SPI基础上进行改造。因为java的SPI需要加载文件中所有扩展点的实现，

java SPI是需要加载文件中所有的实现类，会造成资源浪费，且不能动态加载某个实现类
。

而dubbo的spi文件首先拓展了三个目录下：
- META-INF/services/ 目录：该目录下的 SPI 配置文件用来兼容 JDK SPI 。
- META-INF/dubbo/ 目录：该目录用于存放用户自定义 SPI 配置文件。
- META-INF/dubbo/internal/ 目录：该目录用于存放 Dubbo 内部使用的 SPI 配置文件。

其次dubbo的SPI文件的配置改为了KV形式，实现了只加载对应key的值的扩展点具体实现。

配置文件举例：
```
dubbo=org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
```
这时因为扩展点的key：extensionName为dubbo，所以找到DubboProtocol这个Protocol接口的实现。

### 1. @Spi注解

Dubbo中接口用@SPI注解注释时，标注为扩展点接口，其value值的作用是在加载Protocol接口实现时，如果没有明确指定扩展名，则取value值作为扩展名去加载spi文件中对应的实现类。

#### ExtensionLoader如何处理@SPI注解

ExtensionLoader是SPI实现的核心工具类，对于每个扩展点接口都会有一个ExtensionLoader实例。同时还有一些静态字段作为缓存加载过的扩展点实现。

静态字段：
- strategies(LoadingStrategy[]类型)：LoadingStrategy接口有三个实现，对应是三个SPI配置文件的加载路径。其也都实现了优先级接口，优先级为：
```
 DubboInternalLoadingStrategy > DubboLoadingStrategy > ServicesLoadingStrateg
```
- EXTENSION_LOADERS (ConcurrentHashMap<Class<?>, ExtensionLoader<?>>类型)：表示type类型对应的extensionLoader实例映射缓存。
- EXTENSION_INSTANCES：表示扩展实现类和其实例对象的映射

实例字段：
- type：当前的ExtensionLoader实例负责加载的扩展接口
- objectFactory：所属的对象工厂，注入所依赖的其他扩展点接口时所用
- cachedDefaultName(String类型)：默认扩展名。即@SPI接口的value值
- cachedNames (ConcurrentHashMap<Class<?>, String>类型)：缓存了该ExtensionLoader实例加载的扩展实现类与扩展名之间的映射关系。
- cachedClasses (Holder<ConcurrentHashMap<String, Class<?>>>类型)：缓存了扩展点名称和扩展点实现类之间的映射关系
- cachedInstances (ConcurrentMap<String, Holder<Objcet>>类型)：缓存了该ExtensionLoader实例加载的扩展名与扩展实现对象之间的映射关系。
- cachedAdaptiveInstance：缓存了adaptive扩展点实例
- cachedAdaptiveClass：缓存该extensionLoader加载过程中直接标注@Adaptive注解的扩展实现类
- cachedWrapperClasses：缓存该extensionLoader加载过程中的包装类wrapper实现

ExtensionLoader.getExtensionLoader() 方法会创建对应的ExtensionLoader对象：
```
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        // 从缓存中找 
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
        // 如果缓存为null 则初始化一个 再放入缓存
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
```
这里的new ExtensionLoader就是初始化type和objectFactory两个字段。

初始化完ExtensionLoader实例之后，可以根据getExtension方法用name加载扩展点：
```
public T getExtension(String name) { 
    // getOrCreateHolder()方法中封装了查找cachedInstances缓存的逻辑 
    Holder<Object> holder = getOrCreateHolder(name); 
    Object instance = holder.get(); 
    if (instance == null) { // double-check防止并发问题 
        synchronized (holder) { 
            instance = holder.get(); 
            if (instance == null) { 
                // 根据扩展名从SPI配置文件中查找对应的扩展实现类 
                instance = createExtension(name); 
                holder.set(instance); 
            } 
        } 
    } 
    return (T) instance; 
}

// getOrCreateHolder方法：
    private Holder<Object> getOrCreateHolder(String name) {
        Holder<Object> holder = cachedInstances.get(name);
        if (holder == null) {
            cachedInstances.putIfAbsent(name, new Holder<>());
            holder = cachedInstances.get(name);
        }
        return holder;
    }
    
```

看到是调用了createExtension(name) 来实例化扩展点实现类：
```
/**
     * 获取 cachedClasses 缓存，根据扩展名从 cachedClasses 缓存中获取扩展实现类。
     * 如果 cachedClasses 未初始化，则会扫描前面介绍的三个 SPI 目录获取查找相应的 SPI 配置文件，
     * 然后加载其中的扩展实现类，最后将扩展名和扩展实现类的映射关系记录到 cachedClasses 缓存中。
     * 这部分逻辑在 loadExtensionClasses() 和 loadDirectory() 方法中。
     *
     * 根据扩展实现类从 EXTENSION_INSTANCES 缓存中查找相应的实例。如果查找失败，会通过反射创建扩展实现对象。
     *
     * 自动装配扩展实现对象中的属性（即调用其 setter）。这里涉及 ExtensionFactory 以及自动装配的相关内容，
     *
     * 自动包装扩展实现对象。这里涉及 Wrapper 类以及自动包装特性的相关内容
     *
     * 如果扩展实现类实现了 Lifecycle 接口，在 initExtension() 方法中会调用 initialize() 方法进行初始化。
     * @param name
     * @return
     */
    @SuppressWarnings("unchecked")
    private T createExtension(String name) {
        // 获取扩展名对应的扩展实现类
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            // 从扩展实现类和其实例对象中获取
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                // newInstance放入缓存中
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            // 处理依赖的其他扩展点实现 调用了setter方法
            injectExtension(instance);

            // 实现wrapper包装
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (CollectionUtils.isNotEmpty(wrapperClasses)) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    // 遍历全部wrapper类包装到当前的扩展点实现
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            // 初始化instanced对象 如果扩展点实现了LifeCycle接口的话
            initExtension(instance);
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                    type + ") couldn't be instantiated: " + t.getMessage(), t);
        }
    }
```

### 2. @Adaptive注解和适配器
@Adaptive表示自适应的扩展点实现。
- 当@Adaptive注解在类上注解时，表示此类为扩展实现，在SPI中使用并不多，只使用在了ExtensionFactory接口上。

可以看到ExtensionFactory接口的自适应实现AdaptiveExtensionFactory即为ExtensionLoader.getAdaptiveExtension()方法的返回值，即注解在类上即为自适应实现，且会缓存在ExtensionLoader实例中的cachedAdaptiveClass变量中。

AdaptiveExtensionFactory内部逻辑也比较简单，即根据注入的其他两个ExtensionFactory具体实现去加载对应的扩展点实现。一个是Dubbo SPI自适应扩展点实现加载，一个是Spring上下文获取bean。

- 当@Adaptive注解在方法上时，Dubbo会动态代理生成Adaptive实现类（比如Transporter$Adaptive)，此动态代理类也会实现扩展点接口。

代理类中的逻辑也是根据@Adaptive注解中值作为从url获取扩展名称的key，然后再根据ExtensionLoader获取扩展实现类。

```
public class Transporter$Adaptive implements Transporter { 
    public org.apache.dubbo.remoting.Client connect(URL arg0, ChannelHandler arg1) throws RemotingException { 
        // 必须传递URL参数 
        if (arg0 == null) throw new IllegalArgumentException("url == null"); 
        URL url = arg0; 
        // 确定扩展名，优先从URL中的client参数获取，其次是transporter参数 
        // 这两个参数名称由@Adaptive注解指定，最后是@SPI注解中的默认值 
        String extName = url.getParameter("client",
            url.getParameter("transporter", "netty")); 
        if (extName == null) 
            throw new IllegalStateException("..."); 
        // 通过ExtensionLoader加载Transporter接口的指定扩展实现 
        Transporter extension = (Transporter) ExtensionLoader 
              .getExtensionLoader(Transporter.class) 
                    .getExtension(extName); 
        return extension.connect(arg0, arg1); 
    } 
    ... // 省略bind()方法 
}
```

以上这两种为自适应的适配器实现。获取适配器的代码为getAdaptiveExtension()方法：
```
public T getAdaptiveExtension() {
        // 从缓存中取
        Object instance = cachedAdaptiveInstance.get();
        if (instance == null) {
            if (createAdaptiveInstanceError != null) {
                throw new IllegalStateException("Failed to create adaptive instance: " +
                        createAdaptiveInstanceError.toString(),
                        createAdaptiveInstanceError);
            }

            synchronized (cachedAdaptiveInstance) { // double check
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        // 创建自适应的扩展点实现
                        instance = createAdaptiveExtension();
                        // 缓存
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                    }
                }
            }
        }
        
// createAdaptiveExtension方法中调用的getAdaptiveExtensionClass方法
 private Class<?> getAdaptiveExtensionClass() {
        // 触发loadClass 内部如果有直接加了@Adaptive注解的扩展点实现，则会维护到cacheAdaptiveClass变量中
        getExtensionClasses();
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
        // 动态代理类选择出的扩展点实现也维护在cacheAdaptiveClass变量中
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }

```
- 

### 3. 自动包装特性
扩展点的实现中可能有很多通用逻辑，dubbo SPI中的自动包装特性将多个扩展实现类的公共逻辑抽象到Wrapper类中，Wrapper类和普通扩展点实现一样，也实现了扩展接口，在获取真正的扩展对象时，在外面包一层Wrapper对象，是装饰器模式的实现。

判断是否为Wrapper的实现：
```
private boolean isWrapperClass(Class<?> clazz) {
        try {
            // 检查是否是wrapper包装方式：是否有当前扩展点接口为参数的构造函数
            // wrap类是为了解决多个扩展点实现的公共逻辑
            clazz.getConstructor(type);
            return true;
        } catch (NoSuchMethodException e) {
            return false;
        }
    }
```

在加载spi文件时，会去缓存当前扩展点的wrapper实现类到一个集合变量中：cachedWrapperClasses，然后在之后遍历wrapper实现包装：
```
Set<Class<?>> wrapperClasses = cachedWrapperClasses;
if (CollectionUtils.isNotEmpty(wrapperClasses)) { 
    for (Class<?> wrapperClass : wrapperClasses) { 
        instance = injectExtension((T) wrapperClass 
            .getConstructor(type).newInstance(instance)); 
    } 
}
```


### 4. 自动装配特性
自动装配在injectExtension()方法中。其会扫描所有setter方法，并根据setter方法的名称以及参数的类型，加载相应的扩展实现，然后反射调用setter方法填充属性。

```
private T injectExtension(T instance) { 
    if (objectFactory == null) { // 检测objectFactory字段 
        return instance; 
    } 

    for (Method method : instance.getClass().getMethods()) { 
        ... // 如果不是setter方法，忽略该方法(略) 
        if (method.getAnnotation(DisableInject.class) != null) { 
            continue; // 如果方法上明确标注了@DisableInject注解，忽略该方法 
        } 
        // 根据setter方法的参数，确定扩展接口 
        Class<?> pt = method.getParameterTypes()[0]; 
        ... // 如果参数为简单类型，忽略该setter方法(略) 
        // 根据setter方法的名称确定属性名称 
        String property = getSetterProperty(method); 
        // 加载并实例化扩展实现类 
        Object object = objectFactory.getExtension(pt, property); 
        if (object != null) { 
            method.invoke(instance, object); // 调用setter方法进行装配 
        } 
    } 
    return instance; 
}
```

### 5. @Activate注解和自动激活特性
以Filter接口为例，扩展点的实现非常多，不同场景下需要不同Filter一起执行，根据配置决定哪些场景下哪些Filter自动激活且加入到拦截链中就是@Activate注解的作用。

- group：是Provider端还是Consumer端的
- value：修饰的实现类只在URL参数指定key时才会激活
- order：排序

#### 对@Activate注解的扫描
在loadClass对自动激活的注解进行扫描：
```
private void loadClass(){ 
    if (clazz.isAnnotationPresent(Adaptive.class)) { 
        // 处理@Adaptive注解 
        cacheAdaptiveClass(clazz, overridden); 
    } else if (isWrapperClass(clazz)) { // 处理Wrapper类 
        cacheWrapperClass(clazz); 
    } else { // 处理真正的扩展实现类 
        clazz.getConstructor(); // 扩展实现类必须有无参构造函数 
        ...// 兜底:SPI配置文件中未指定扩展名称，则用类的简单名称作为扩展名(略) 
        String[] names = NAME_SEPARATOR.split(name); 
        if (ArrayUtils.isNotEmpty(names)) { 
            // 将包含@Activate注解的实现类缓存到cachedActivates集合中 
            cacheActivateClass(clazz, names[0]); 
            for (String n : names) { 
                // 在cachedNames集合中缓存实现类->扩展名的映射 
                cacheName(clazz, n);
                // 在cachedClasses集合中缓存扩展名->实现类的映射 
                saveInExtensionClass(extensionClasses, clazz, n, 
                     overridden); 
            } 
        } 
    } 
}
```
#### getActivateExtension方法
在此方法中：
- 如果传入配置没有-default配置，会触发写入自动激活的缓存。
- 然后遍历需要自动激活的扩展点接口，如果符合group（provider端或consumer端），并且没有出现在names配置的和被去除的，则加载激活扩展点实现，放入到activateExtensions且sort方法排序。
- 遍历传入的filter配置（这里和配置文件的顺序保持一致），会处理与default扩展点实现（上一步加载的自动激活扩展点实现）的顺序和配置保持一致。
```
public List<T> getActivateExtension(URL url, String[] values, String group) {
        List<T> activateExtensions = new ArrayList<>();
        // names是dubbo配置传入的顺序
        List<String> names = values == null ? new ArrayList<>(0) : asList(values);
        if (!names.contains(REMOVE_VALUE_PREFIX + DEFAULT_KEY)) { // 无-default
            // 触发 cachedActivate缓存字段的加载
            getExtensionClasses();
            for (Map.Entry<String, Object> entry : cachedActivates.entrySet()) {
                String name = entry.getKey(); // 扩展名
                Object activate = entry.getValue(); // @Activate注解

                String[] activateGroup, activateValue;

                if (activate instanceof Activate) {
                    activateGroup = ((Activate) activate).group();
                    activateValue = ((Activate) activate).value();
                } else if (activate instanceof com.alibaba.dubbo.common.extension.Activate) {
                    activateGroup = ((com.alibaba.dubbo.common.extension.Activate) activate).group();
                    activateValue = ((com.alibaba.dubbo.common.extension.Activate) activate).value();
                } else {
                    continue;
                }
                if (isMatchGroup(group, activateGroup) // 匹配group
                        && !names.contains(name) // 没有出现在names中 走默认激活的
                        && !names.contains(REMOVE_VALUE_PREFIX + name) // 未在配置中去除的
                        && isActive(activateValue, url)) { // 检查url中是否出现指定的key
                    // 加载扩展实现的实例对象 这些是不在传入的 names 里的且被去除掉的
                    activateExtensions.add(getExtension(name));
                }
            }
            // 对不在filter配置中加载的激活扩展点进行排序
            activateExtensions.sort(ActivateComparator.COMPARATOR);
        }
        List<T> loadedExtensions = new ArrayList<>();
        for (int i = 0; i < names.size(); i++) {
            String name = names.get(i);
            if (!name.startsWith(REMOVE_VALUE_PREFIX) // -开头的不加载 -开头的直接跳过 因为在上边已经过滤了
                    && !names.contains(REMOVE_VALUE_PREFIX + name)) {
                if (DEFAULT_KEY.equals(name)) {
                    // 这里在default之前的都会放在上边加载过的默认自动激活扩展点之前
                    if (!loadedExtensions.isEmpty()) {
                        // 按照顺序 把自定义的扩展添加 到 默认扩展集合之前
                        activateExtensions.addAll(0, loadedExtensions);
                        loadedExtensions.clear();
                    }
                } else {
                    // 根据扩展名去加载对应的扩展点实现类
                    loadedExtensions.add(getExtension(name));
                }
            }
        }
        if (!loadedExtensions.isEmpty()) {
            // 在default之后的会加载到default之后
            activateExtensions.addAll(loadedExtensions);
        }
        return activateExtensions;
    }
```

比如有如下几个Filter
![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/mac_picgo/20210714192712.png)

传入filter配置是为Provider端的："demoFilter3、-demoFilter2、default、demoFilter1"。
那么最终Filter链的结果是： [demoFilter3, demoFilter6, demoFilter4, demoFilter1]。

