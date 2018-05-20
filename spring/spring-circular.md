

aop,这里如果beanPostProcess没有修改bean的类，那么就返回early创建的对象
所以如果要做加强
如果被依赖了，要在proxy生成代理类,然后在early的地方返回
然后beanPostProcess的时候不会再修改bean了

但是如果没有依赖，则proxy会在beanPostProcess的after里把bean替换成proxy

Object earlySingletonReference = getSingleton(beanName, false);
if (earlySingletonReference != null) {
    if (exposedObject == bean) {
        exposedObject = earlySingletonReference;
    }
}
所以：
beanPostProcess 
before
都是原对象

after
可能拿到proxy(被aop增强了)
也可能是原对象(循环引用时，这里会时一个原对象)