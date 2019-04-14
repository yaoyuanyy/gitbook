# BeanPostProcessor集团的解析
BeanPostProcessor是spring框架中非常重要的类，每一个放到spring ioc中的bean的创建过程都有他的身影。俗称前置处理和后置处理，意思为在bean的创建前后可以增加自定义的处理逻辑。如，bean创建前给某个类型的bean增加一个属性值，在bean创建后生成这个bean的代理类放到spring ioc中等等

本文先说下BeanPostProcessor方法被调用的入口，然后再简介重要的BeanPostProcessor实现类及特点

## BeanPostProcessor及实现类被调用入口
 
AbstractBeanFactroy.getBean方法
<font size=2>
-this.getSingleton方法
--DefaultSingletonBeanRegistry.getObject方法
---AbstractAutowireCapableBeanFactroy.createBean方法
----this.doCreateBean方法
-----this.initializeBean方法
------this.invokeAwareMethods方法
------this.applyBeanPostProcessorsBeforeInitialization方法
-------for beanProcessor.postProcessBeforeInitialization方法
------this.applyBeanPostProcessorsAfterInitialization方法
-------for beanProcessor.postProcessAfterInitialization方法
</font>

AbstractAutowireCapableBeanFactroy.initializeBean(String beanName,Object bean, RootBeanDefinition mbd)作为直接调用BeanPostProcessor.applyBeanPostProcessorsXXXInitialization的方法，方法代码如下
```
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
	...
	invokeAwareMethods(beanName, bean);

	Object wrappedBean = bean;
	if (mbd == null || !mbd.isSynthetic()) {
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}
    ... 
	invokeInitMethods(beanName, wrappedBean, mbd);
	...
	if (mbd == null || !mbd.isSynthetic()) {
		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
	}
	return wrappedBean;
}
```
## 家族各个子类

### BeanPostProcessor
TODO

### InstantiationAwareBeanPostProcessor


注意： BeanPostProcessor是在spring容器加载了bean的定义文件并且实例化bean之后执行的。 BeanPostProcessor的执行顺序是在BeanFactoryPostProcessor之后。

Spring中，有内置的一些BeanPostProcessor实现类，例如：

org.springframework.context.annotation.CommonAnnotationBeanPostProcessor：支持@Resource注解的注入
org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor：支持@Required注解的注入
org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor：支持@Autowired注解的注入
org.springframework.orm.jpa.support.PersistenceAnnotationBeanPostProcessor：支持@PersistenceUnit和@PersistenceContext注解的注入
org.springframework.context.support.ApplicationContextAwareProcessor：用来为bean注入ApplicationContext等容器对象
