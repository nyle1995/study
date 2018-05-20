# 浅述 Spring 代理如何在循环依赖中解决的

## 循环依赖
也就是bean A 依赖 beanB，然后bean B 也依赖bean A

这里的解决比较简单,在初始A开始的时候，就放了一个设置了一个预先加载的方法  
```
addSingletonFactory(beanName, new ObjectFactory<Object>() {
    @Override
    public Object getObject() throws BeansException {
        return getEarlyBeanReference(beanName, mbd, bean);
    }
});
```
当B加载的时候，发现A正在加载。就会先加载A的early对象注入自己的参数里
```
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

## 代理在循环依赖中的解决

aop相关的代理，相当于会把bean替换成一个代理类。  
那看下循环依赖的时候怎么处理

主要逻辑在AbstractAutoProxyCreator中  
实现了getEarlyBeanReference,会在这里直接返回一个代理类  
然后放到earlyProxyReferences, 标记这个bean已经被early处理过了
```
@Override
public Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    if (!this.earlyProxyReferences.contains(cacheKey)) {
        this.earlyProxyReferences.add(cacheKey);
    }
    return wrapIfNecessary(bean, beanName, cacheKey);
}
```

然后在postProcessAfterInitialization，会判定如果在early处理过了，就不需要在进行代理了
```
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (!this.earlyProxyReferences.contains(cacheKey)) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
```

这里又有几个其他的问题:  
1. 既然beanPostProcessor没有替换调bean成代理类，那怎么把代理类放到beanFactory里的
2. 如果有其他的BeanPostProcessor把bean换掉了，导致代理的对象变了, 这怎么感知到  

spring在初始化完之后，会check一遍  
如果有循环依赖，即有early对象了
    - 而且bean没有在beanPostProcessor被替换掉，那么就返回early对象到容器里
    - 然后被修改过了，就会抛错处理
```
bject earlySingletonReference = getSingleton(beanName, false);
if (earlySingletonReference != null) {
    // 如果有循环依赖，即有early对象了。而且bean没有在beanPostProcessor被替换掉，那么就返回early到容器里
    if (exposedObject == bean) {
        exposedObject = earlySingletonReference;
    }
    else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
        String[] dependentBeans = getDependentBeans(beanName);
        Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
        for (String dependentBean : dependentBeans) {
            if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                actualDependentBeans.add(dependentBean);
            }
        }
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
```

## 其他
所以这里可以发现一个其他的问题
如果你自己实现了一个beanPostProcessor, 那么你在postProcessAfterInitialization拿到的可能是代理类，也可能是原对象。  
取决于这个对象是否被循环依赖了
