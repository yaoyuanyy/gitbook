# spring boot AspectJ注解处理过程分析

## 前提
springboot对AspectJ注解的处理是用过AnnotationAwareAspectJAutoProxyCreator完成的。
首先看下AutoProxyCreator类层级结构
![AutoProxyCreator子类2](media/AutoProxyCreator%E5%AD%90%E7%B1%BB2.png)


![AutoProxyCreator子类](media/AutoProxyCreator%E5%AD%90%E7%B1%BB.png)



## 项目启动时初始化处理AspectJ注解相关类的分析
springboot之所以能实现自动配置，是因为他的springboot-autoconfigure.jar包有个spring.factores文件，里面记录着很多的XXXAutoConfiguration类
![-w677](media/15539502575727.jpg)
可以找到AopAutoConfiguration类，类实现如下
```
@Configuration
@ConditionalOnClass({ EnableAspectJAutoProxy.class, Aspect.class, Advice.class })
@ConditionalOnProperty(prefix = "spring.aop", name = "auto", havingValue = "true", matchIfMissing = true)
public class AopAutoConfiguration {

	@Configuration
	@EnableAspectJAutoProxy(proxyTargetClass = false)
	@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false", matchIfMissing = true)
	public static class JdkDynamicAutoProxyConfiguration {

	}

	@Configuration
	@EnableAspectJAutoProxy(proxyTargetClass = true)
	@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true", matchIfMissing = false)
	public static class CglibAutoProxyConfiguration {

	}

}
```
这里可以知道默认情况下使用Jdk动态代理，因为@ConditionalOnProperty的属性决定了使用哪种方式代理.
EnableAspectJAutoProxy是代理相关功能的入口，他的代码如下
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
	boolean proxyTargetClass() default false;
	boolean exposeProxy() default false;
}
```
我们知道，注解的作用是打标识，真正起作用的的是@Import的AspectJAutoProxyRegistrar.class，springboot解析@Configuration类的会解析@Import注解，从而加载AspectJAutoProxyRegistrar类。所以关键地方在AspectJAutoProxyRegistrar，看代码实现
```
/**
 * 根据给定的@EnableAspectJAutoProxy注释，注册一个AnnotationAwareAspectJAutoProxyCreator给当前的BeanDefinitionRegistry 
 */
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

	/**
	 * 根据@Configuration class import的属性EnableAspectJAutoProxy#proxyTargetClass()的值去注册、配置AspectJ auto proxy creator based on the value
	 * /
	@Override
	public void registerBeanDefinitions(
			AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

		AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry); // (1)

		AnnotationAttributes enableAspectJAutoProxy =
				AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
		if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
			AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry); // (2)
		}
		if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
			AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);// (3)
		}
	}

}
```
(1)处代码是注册一个AnnotationAwareAspectJAutoProxyCreator到BeanFactory容器中。(2)处根据proxyTargetClass的值判断是否使用cglib的代理方式(这就是我们日常工作中要想使用cglib代理方式时声明proxyTargetClass=true的原因)。(3)处代码的作用是否可以通过AopContext.currentProxy()取到代理对象。

到这里有关aop初始化的部分就完了。最终，AnnotationAwareAspectJAutoProxyCreator bean被创建后放入beanFactory。后面当其他类bean instance时会用到它，为什么会用到它呢？因为<font size=4 color=green>核心关键点：</font>AnnotationAwareAspectJAutoProxyCreator是一个BeanPostProcessor，所以beanFactory容器中所有的类实例化时都会经过他的的前置处理，根据条件或规则看看是否需要生成代理类

## 代理类创建入口<font size=4>-AnnotationAwareAspectJAutoProxyCreator发挥威力之时</font>
因为AnnotationAwareAspectJAutoProxyCreator是InstantiationAwareBeanPostProcessor与BeanPostProcessor共同子类，又因为XXXBeanPostProcessors会在beanClass(bean definition)被创建bean之前使用实例化前置处理(postProcessBeforeInstantiation方法)，所以每个beanClass被创建bean之前都会通过AnnotationAwareAspectJAutoProxyCreator的实例化前置处理。而这个实例化前置处理方法逻辑就是为符合条件的beanClass(bean definition)生成proxy代理类。实例化前置方法是继承父类AbstractAutoProxyCreator的，方法的代码如下
<font size=2>AbstractAutoProxyCreator#postProcessBeforeInstantiation方法</font>
```
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
	Object cacheKey = getCacheKey(beanClass, beanName);

	if (beanName == null || !this.targetSourcedBeans.contains(beanName)) {
		if (this.advisedBeans.containsKey(cacheKey)) {
			return null;
		}
		if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return null;
		}
	}

	// Create proxy here if we have a custom TargetSource
	if (beanName != null) {
		TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
		if (targetSource != null) {
			this.targetSourcedBeans.add(beanName);
			Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
			Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}
	}

	return null;
}

```
方法通过滤重和条件判断，符合条件的bean会被创建代理。条件判断是重点，isInfrastructureClass方法和shouldSkip方法共同作用。isInfrastructureClass(beanClass)判断beanClass如果是advice,advisor,PointCut,AopInfrastructureBean子类或者beanClass有Aspect注解，这些情况都不产生代理；如果是false，还要关注shouldSkip(beanClass, beanName)的返回值，shouldSkip方法的作用是找出beanFactory容器里的advosor beans,如果有advosor.getAdvice().getName等于beanClass.name,那么也不产生代理


