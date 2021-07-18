---
title: '@Async加循环依赖的启动报错问题'
copyright: true
date: 2020-03-18 23:26:16
tags: 
	- @Async 
	- 循环依赖
categories:
	- spring
	- spring boot
---

# 现象及相关引用博客
https://segmentfault.com/a/1190000021217176
# 问题
Spring其实是可以帮助解决循环依赖的，但是在循环依赖的两个bean上有一个加入了@Async注解之后，在启动的时候就报错不能进行循环依赖。

```
@Component
public class A {

    @Autowired
    private B b;

    @Async
    public void testA() {
        System.out.println(Thread.currentThread().getName());
    }
}


@Component
public class B {
    @Autowired
    private A a;

    public void testB() {
        System.out.println("调用到了B");
    }
}
```
对应的错误：
```
org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'a': Bean with name 'a' has been injected into other beans [b] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using 'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.
```

注意这里@Transaction虽然也是使用的代理，但是循环引用如果是@Transaction注解 是不影响启动的 可以在最早初始化类实例的时候就能拿到代理对象， 而async是在postProcessor后置处理器当中处理的，所以在循环引用时会放入原始对象而不是代理对象 在之后的check时会报错。这里要做下区分。


# 问题分析及解决方案
报错所在方法：
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean

```
protected Object doCreateBean( ... ){
...
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences && isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
}
...

// populateBean这一句特别的关键，它需要给A的属性赋值，所以此处会去实例化B~~
// 而B我们从上可以看到它就是个普通的Bean（并不需要创建代理对象），实例化完成之后，继续给他的属性A赋值，而此时它会去拿到A的早期引用
// 也就在此处在给B的属性a赋值的时候，会执行到上面放进去的Bean A流程中的getEarlyBeanReference()方法  从而拿到A的早期引用~~
// 执行A的getEarlyBeanReference()方法的时候，会执行自动代理创建器，但是由于A没有标注事务，所以最终不会创建代理，so B合格属性引用会是A的**原始对象**
// 需要注意的是：@Async的代理对象不是在getEarlyBeanReference()中创建的，是在postProcessAfterInitialization创建的代理
// 从这我们也可以看出@Async的代理它默认并不支持你去循环引用，因为它并没有把代理对象的早期引用提供出来~~~（注意这点和自动代理创建器的区别~）

// 结论：此处给A的依赖属性字段B赋值为了B的实例(因为B不需要创建代理，所以就是原始对象)
// 而此处实例B里面依赖的A注入的仍旧为Bean A的普通实例对象（注意  是原始对象非代理对象）  注：此时exposedObject也依旧为原始对象
populateBean(beanName, mbd, instanceWrapper);

// 标注有@Async的Bean的代理对象在此处会被生成~~~ 参照类：AsyncAnnotationBeanPostProcessor
// 所以此句执行完成后  exposedObject就会是个代理对象而非原始对象了
exposedObject = initializeBean(beanName, exposedObject, mbd);

...
// 这里是报错的重点~~~
if (earlySingletonExposure) {
    // 上面说了A被B循环依赖进去了，所以此时A是被放进了二级缓存的，所以此处earlySingletonReference 是A的原始对象的引用
    // （这也就解释了为何我说：如果A没有被循环依赖，是不会报错不会有问题的   因为若没有循环依赖earlySingletonReference =null后面就直接return了）
    Object earlySingletonReference = getSingleton(beanName, false);
    if (earlySingletonReference != null) {
        // 上面分析了exposedObject 是被@Aysnc代理过的对象， 而bean是原始对象 所以此处不相等  走else逻辑
        if (exposedObject == bean) {
            exposedObject = earlySingletonReference;
        }
        // allowRawInjectionDespiteWrapping 标注是否允许此Bean的原始类型被注入到其它Bean里面，即使自己最终会被包装（代理）
        // 默认是false表示不允许，如果改为true表示允许，就不会报错啦。这是我们后面讲的决方案的其中一个方案~~~
        // 另外dependentBeanMap记录着每个Bean它所依赖的Bean的Map~~~~
        else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
            // 我们的Bean A依赖于B，so此处值为["b"]
            String[] dependentBeans = getDependentBeans(beanName);
            Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);

            // 对所有的依赖进行一一检查~    比如此处B就会有问题
            // “b”它经过removeSingletonIfCreatedForTypeCheckOnly最终返返回false  因为alreadyCreated里面已经有它了表示B已经完全创建完成了~~~
            // 而b都完成了，所以属性a也赋值完成儿聊 但是B里面引用的a和主流程我这个A竟然不相等，那肯定就有问题(说明不是最终的)~~~
            // so最终会被加入到actualDependentBeans里面去，表示A真正的依赖~~~
            for (String dependentBean : dependentBeans) {
                if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                    actualDependentBeans.add(dependentBean);
                }
            }

            // 若存在这种真正的依赖，那就报错了~~~  则个异常就是上面看到的异常信息
            if (!actualDependentBeans.isEmpty()) {
                throw new BeanCurrentlyInCreationException(beanName,
                        "Bean with name '" + beanName + "' has been injected into other beans [" +
                        StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                        "] in its raw version as part of a circular reference, but has eventually been " +
                        "wrapped. This means that said other beans do not use the final version of the " +
                        "bean. This is often the result of over-eager type matching - consider using " +
                        "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
            }
        }
    }
}
...
}
```

debug看到的对象注入：
![](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/mac_picgo/20210718223108.png)

可以看到@Async注解标注的Bean的创建代理的时机是在检查bean中引用的之后的。看@EnableAsync注解会通过AsyncConfigurationSelector注入AsyncAnnotationBeanPostProcessor这个后置处理器，在其实现了postProcessAfterInitalization方法，创建代理即在此中。

这里的根本原理是只要能被切面AsyncAnnotationAdvisor切入的Bean都会在后置处理器中生成一个代理对象（如果已经是代理对象，那么加入该切面即可），赋值为上边doCreateBean中的exposedObject作为返回值加入到spring容器中。

```
// 关键是这里。当Bean初始化完成后这里会执行，这里会决策看看要不要对此Bean创建代理对象再返回~~~
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
    if (this.advisor == null || bean instanceof AopInfrastructureBean) {
        // Ignore AOP infrastructure such as scoped proxies.
        return bean;
    }

    // 如果此Bean已经被代理了（比如已经被事务那边给代理了~~）
    if (bean instanceof Advised) {
        Advised advised = (Advised) bean;
    
        // 此处拿的是AopUtils.getTargetClass(bean)目标对象，做最终的判断
        // isEligible()是否合适的判断方法  是本文最重要的一个方法，下文解释~
        // 此处还有个小细节：isFrozen为false也就是还没被冻结的时候，就只向里面添加一个切面接口   并不要自己再创建代理对象了  省事
        if (!advised.isFrozen() && isEligible(AopUtils.getTargetClass(bean))) {
            // Add our local Advisor to the existing proxy's Advisor chain...
            // beforeExistingAdvisors决定这该advisor最先执行还是最后执行
            // 此处的advisor为：AsyncAnnotationAdvisor  它切入Class和Method标注有@Aysnc注解的地方~~~
            if (this.beforeExistingAdvisors) {
                advised.addAdvisor(0, this.advisor);
            } else {
                advised.addAdvisor(this.advisor);
            }
            return bean;
        }
    }

    // 若不是代理对象，此处就要下手了~~~~isEligible() 这个方法特别重要
    if (isEligible(bean, beanName)) {
        // copy属性  proxyFactory.copyFrom(this); 生成一个新的ProxyFactory 
        ProxyFactory proxyFactory = prepareProxyFactory(bean, beanName);
        // 如果没有强制采用CGLIB 去探测它的接口~
        if (!proxyFactory.isProxyTargetClass()) {
            evaluateProxyInterfaces(bean.getClass(), proxyFactory);
        }
        // 添加进此切面~~ 最终为它创建一个getProxy 代理对象
        proxyFactory.addAdvisor(this.advisor);
        //customize交给子类复写（实际子类目前都没有复写~）
        customizeProxyFactory(proxyFactory);
        return proxyFactory.getProxy(getProxyClassLoader());
    }

    // No proxy needed.
    return bean;
}

// 我们发现BeanName最终其实是没有用到的~~~
// 但是子类AbstractBeanFactoryAwareAdvisingPostProcessor是用到了的  没有做什么 可以忽略~~~
protected boolean isEligible(Object bean, String beanName) {
    return isEligible(bean.getClass());
}
protected boolean isEligible(Class<?> targetClass) {
    // 首次进来eligible的值肯定为null~~~
    Boolean eligible = this.eligibleBeans.get(targetClass);
    if (eligible != null) {
        return eligible;
    }
    // 如果根本就没有配置advisor  也就不用看了~
    if (this.advisor == null) {
        return false;
    }
    
    // 最关键的就是canApply这个方法，如果AsyncAnnotationAdvisor  能切进它  那这里就是true
    // 本例中方法标注有@Aysnc注解，所以铁定是能被切入的  返回true继续上面方法体的内容
    eligible = AopUtils.canApply(this.advisor, targetClass);
    this.eligibleBeans.put(targetClass, eligible);
    return eligible;
}
```