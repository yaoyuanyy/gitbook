# BeanFactoryPostProcessory与BeanPostProcessor区别
<font color=gray size=2>时间: 2019-03-24</font>

BeanFactoryPostProcessory与BeanPostProcessor在spring框架中扮演着很重要的角色。只要你debug源码，你都可以发现他们的身影。在项目启动时，BeanFactoryPostProcessory比BeanPostProcessor先实例化和初始化。通过他们的各个简介，我们既能了解他们自身，也能够掌握他们的区别所在

## BeanFactoryPostProcessory简介
首先看下类的javadoc 
<font size=1>
* Allows for custom modification of an application context's bean definitions, adapting the bean property values of the context's underlying bean factory.
* Application contexts can auto-detect BeanFactoryPostProcessor beans in their bean definitions and apply them before any other beans get created.
* Useful for custom config files targeted at system administrators that override bean properties configured in the application context.
* See PropertyResourceConfigurer and its concrete implementations for out-of-the-box solutions that address such configuration needs.
* A BeanFactoryPostProcessor may interact with and modify bean definitions, but never bean instances. Doing so may cause premature bean instantiation, violating the container and causing unintended side-effects. If bean instance interaction is required, consider implementing BeanPostProcessor instead

</font>

翻译如下：
<font size=1>
* 允许application context的bean definitions自定义修改，改写the context的底层bean factory的bean的属性值。
* Application context能通过他的bean definitions自动检测到 BeanFactoryPostProcessor beans，并在其他的beans被创建前应用the beans
* 自定义config files去覆盖application context配置的bean属性是有用的
* 查看PropertyResourceConfigurer类和他的具体的实现为解决那些配置需要的即插即用的方案 // todo 没懂
* A BeanFactoryPostProcessor用于修改bean definitions，而不是bean instances。这么做可导致bean过早的 instantiation(实例化)、侵犯容器和其他意外的副作用。如果你要bean instance，请使用BeanPostProcessor

</font>

<font color="#FF0000">注</font>: 最后一段也是<font color="green" size=3 face="宋体"> *BeanFactoryPostProcessory与BeanPostProcessor区别*</font>

### 方法
他的方法只有一个，通过这个方法可以做到：bean覆盖或添加属性，甚至是初始化bean
```
/**
 * 允许覆盖或添加属性，甚至是初始化bean
 */
void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

```
## BeanPostProcessor简介
首先看下类的javadoc 
<font size=1>
* Factory hook that allows for custom modification of new bean instances, e.g. checking for marker interfaces or wrapping them with proxies.
* ApplicationContexts can autodetect BeanPostProcessor beans in their bean definitions and apply them to any beans subsequently created. Plain bean factories allow for programmatic registration of post-processors, applying to all beans created through this factory
* Typically, post-processors that populate beans via marker interfaces or the like will implement postProcessBeforeInitialization(java.lang.Object, java.lang.String), while post-processors that wrap beans with proxies will normally implement postProcessAfterInitialization(java.lang.Object, java.lang.String)

</font>

翻译如下：
<font size=1>
* BeanPostProcessor是一个 Factory hook，他允许自定义bean instances的修改，例如：检查marker interfaces 或者wrapping them with proxies
* Application contexts能在他们的bean definitions中自动检查到BeanPostProcessror beans，并在随后创建的任何beans中应用他们。普通的bean factorys允许post-processors的程序化注册，通过this factory应用到所有创建的beans
* 典型的，通过marker interfaces或者类似的产生beans的post-processors将实现postProcessBeforInitialization(Object,String)方法，而通过porxies包装beans的post-processors将实现postProcessAfterInitialization(Object,String)方法

</font>

### 方法
```
前置方法
/**
 * Apply this BeanPostProcessor to the given new bean instance before any bean initialization callbacks (like InitializingBean's afterPropertiesSet or a custom init-method). 
 * The bean will already be populated with property values. The returned bean instance may be a wrapper around the original.
 */
Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;



后置方法
 * Apply this BeanPostProcessor to the given new bean instance after any bean initialization callbacks (like InitializingBean's afterPropertiesSet or a custom init-method). The bean will already be populated with property values. The returned bean instance may be a wrapper around the original.
 * In case of a FactoryBean, this callback will be invoked for both the FactoryBean instance and the objects created by the FactoryBean (as of Spring 2.0). The post-processor can decide whether to apply to either the FactoryBean or created objects or both through corresponding bean instanceof FactoryBean checks.
 * This callback will also be invoked after a short-circuiting triggered by a InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation(java.lang.Class<?>, java.lang.String) method, in contrast to all other BeanPostProcessor callbacks
 */
Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

```

## 总结：
- <font color="green">*BeanFactoryPostProcessor负责处理bean definitions. 而BeanPostProcessor负责处理bean instances*</font>
- 由于bean definition先与bean instance，所以spring 框架先应用BeanFactoryPostProcessor，之后再应用BeanPostProcessor
- 通俗的话讲，在bean instance之前，你可以使用BeanFactoryPostProcessor操作bean definitons来编辑bean(覆盖或属性赋值等)，然后BeanPostProcessor进行bean instance操作


以下从stackoverflow获取
#### The differences about BeanFactoryPostProcessor and BeanPostProcessor:
* A bean implementing BeanFactoryPostProcessor is called when all bean definitions will have been loaded, but no beans will have been instantiated yet. This allows for overriding or adding properties even to eager-initializing beans. This will let you have access to all the beans that you have defined in XML or that are annotated (scanned via component-scan).
* A bean implementing BeanPostProcessor operate on bean (or object) instances which means that when the Spring IoC container instantiates a bean instance then BeanPostProcessor interfaces do their work.
* BeanFactoryPostProcessor implementations are "called" during startup of the Spring context after all bean definitions will have been loaded while BeanPostProcessor are "called" when the Spring IoC container instantiates a bean (i.e. during the startup for all the singleton and on demand for the proptotypes one)

## 问题
1. bean definition 与bean instance在哪个地方有联系的。即bean definition什么时候在哪里转化成bean instance的
2. BeanFactoryPostProcessor和BeanPostProcessor被Applicatioin context自动检测到是如何实现的

## 题外 
### BeanFactoryPostProcessor与BeanDefinitionRegistryPostProcessor的区别,见？？