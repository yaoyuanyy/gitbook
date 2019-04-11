# BeanFactoryPostProcessor集团解析
date 2019-03-24

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

#### 方法
他的方法只有一个，通过这个方法可以做到：bean覆盖或添加属性，甚至是初始化bean
```
/**
 * 允许覆盖或添加属性，甚至是初始化bean
 */
void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

```

## BeanDefinitionRegistryPostProcessor简介

他的javadoc
* <font size=1>这个接口扩展了标准的BeanFactoryPostProcessor 接口，允许在普通的BeanFactoryPostProcessor接口实现类执行之前注册更多的BeanDefinition。特别地是，BeanDefinitionRegistryPostProcessor可以注册BeanFactoryPostProcessor的BeanDefinition
</font>

方法
```
/**
 * Modify the application context's internal bean definition registry after its
 * standard initialization. All regular bean definitions will have been loaded,
 * but no beans will have been instantiated yet. This allows for adding further
 * bean definitions before the next post-processing phase kicks in.
 */
void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
```
postProcessBeanDefinitionRegistry方法可以修改在BeanDefinitionRegistry接口实现类中注册的任意BeanDefinition，也可以增加和删除BeanDefinition。原因是这个方法执行前所有常规的BeanDefinition已经被加载到BeanDefinitionRegistry接口实现类中，但还没有bean被实例化。

实际上，Mybatis中org.mybatis.spring.mapper.MapperScannerConfigurer就实现了该方法，在只有接口没有实现类的情况下找到接口方法与sql之间的联系从而生成BeanDefinition并注册。而Spring ConfigurationClassPostProcessor也是用来将注解@Configuration中的相关生成bean的方法所对应的BeanDefinition进行注册。


## PropertySourcesPlaceholderConfigurer
```
// todo
```

## BeanDefinitionRegistryPostProcessor与BeanFactoryPostProcessor的区别
BeanDefinitionRegistryPostProcessor继承BeanFactoryPostProcessor，除了BeanFactoryPostProcessor的功能外，<font color=green size = 4>*BeanDefinitionRegistryPostProcessor可以注册BeanFactoryPostProcessor的BeanDefinition*</font>，这也是他和BeanFactoryPostProcessor的主要区别

## BeanFactoryPostProcessor及实现类方法调用入口
调用入口栈
```
AbstractApplicationContext.refresh()
-- PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors
---- BeanDefinitionRegistryPostProcessors.invokeBeanDefinitionRegistryPostProcessors
---- BeanFactoryPostProcessor.invokeBeanFactoryPostProcessors
```
所以调用BeanFactoryPostProcessor方法的入口在PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors方法。看代码
```
public static void invokeBeanFactoryPostProcessors(
		ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

	// Invoke BeanDefinitionRegistryPostProcessors first, if any.
	Set<String> processedBeans = new HashSet<String>();

	if (beanFactory instanceof BeanDefinitionRegistry) {
		BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
		List<BeanFactoryPostProcessor> regularPostProcessors = new LinkedList<BeanFactoryPostProcessor>();
		List<BeanDefinitionRegistryPostProcessor> registryProcessors = new LinkedList<BeanDefinitionRegistryPostProcessor>();

		for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
			if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
				BeanDefinitionRegistryPostProcessor registryProcessor =
						(BeanDefinitionRegistryPostProcessor) postProcessor;
				registryProcessor.postProcessBeanDefinitionRegistry(registry);
				registryProcessors.add(registryProcessor);
			}
			else {
				regularPostProcessors.add(postProcessor);
			}
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the bean factory post-processors apply to them!
		// Separate between BeanDefinitionRegistryPostProcessors that implement
		// PriorityOrdered, Ordered, and the rest.
		List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<BeanDefinitionRegistryPostProcessor>();

		// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
				processedBeans.add(ppName);
			}
		}
		sortPostProcessors(currentRegistryProcessors, beanFactory);
		registryProcessors.addAll(currentRegistryProcessors);
		invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
		currentRegistryProcessors.clear();

		// Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
		postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
		for (String ppName : postProcessorNames) {
			if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
				currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
				processedBeans.add(ppName);
			}
		}
		sortPostProcessors(currentRegistryProcessors, beanFactory);
		registryProcessors.addAll(currentRegistryProcessors);
		invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
		currentRegistryProcessors.clear();

		// Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
		boolean reiterate = true;
		while (reiterate) {
			reiterate = false;
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (!processedBeans.contains(ppName)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
					reiterate = true;
				}
			}
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();
		}

		// Now, invoke the postProcessBeanFactory callback of all processors handled so far.
		invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
	}

	else {
		// Invoke factory processors registered with the context instance.
		invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
	}

	// Do not initialize FactoryBeans here: We need to leave all regular beans
	// uninitialized to let the bean factory post-processors apply to them!
	String[] postProcessorNames =
			beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

	// Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
	// Ordered, and the rest.
	List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
	List<String> orderedPostProcessorNames = new ArrayList<String>();
	List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
	for (String ppName : postProcessorNames) {
		if (processedBeans.contains(ppName)) {
			// skip - already processed in first phase above
		}
		else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
			priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
		}
		else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
			orderedPostProcessorNames.add(ppName);
		}
		else {
			nonOrderedPostProcessorNames.add(ppName);
		}
	}

	// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
	sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
	invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

	// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
	List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
	for (String postProcessorName : orderedPostProcessorNames) {
		orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
	}
	sortPostProcessors(orderedPostProcessors, beanFactory);
	invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

	// Finally, invoke all other BeanFactoryPostProcessors.
	List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
	for (String postProcessorName : nonOrderedPostProcessorNames) {
		nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
	}
	invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

	// Clear cached merged bean definitions since the post-processors might have
	// modified the original metadata, e.g. replacing placeholders in values...
	beanFactory.clearMetadataCache();
}
```
方法很长，但是逻辑挺简单的，反复就是做一件事：invokeBeanFactoryPostProcessors(List<BeanFactoryPostProcessor>, beanFactory),只不过按一定的顺序维度分别调用的，具体的顺序为
1. 首先，AbstractApplicationContext.beanFactoryPostProcessors属性
2. 其次，实现了PriorityOrdered接口的BeanDefinitionRegistryPostProcessors
3. 然后，实现了ordered接口的BeanDefinitionRegistryPostProcessors
4. 再然后，其他的BeanDefinitionRegistryPostProcessors
5. 最后，常规的BeanFactoryPostProcessors

从这个顺序我们可以知道一个知识点:本文上面的BeanDefinitionRegistryPostProcessor与BeanFactoryPostProcessor的区别；BeanDefinitionRegistryPostProcessor.invokeBeanDefinitionRegistryPostProcessor方法比BeanFactoryPostProcessor.invokeBeanFactoryPostProcessors方法先被调用，
```
# BeanDefinitionRegistryPostProcessor.class
invokeBeanDefinitionRegistryPostProcessor方法可以进一步的处理Bean definition
private static void invokeBeanDefinitionRegistryPostProcessor(
		Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry) {

	for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {
		postProcessor.postProcessBeanDefinitionRegistry(registry);
	}
}

# BeanFactoryPostProcessor.class
invokeBeanFactoryPostProcessors可以覆盖和add属性值，甚至是实例化bean
private static void invokeBeanFactoryPostProcessors(
		Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {

	for (BeanFactoryPostProcessor postProcessor : postProcessors) {
		postProcessor.postProcessBeanFactory(beanFactory);
	}
}
```
下面就是进入各个BeanDefinitionRegistryPostProcessor和BeanFactoryPostProcessor自己的方法走逻辑了，不再详述

下面说下PostProcessorRegistrationDelegate类，他是AbstractApplicationContext的委托类，专门负责post-processor处理，别看这个类代码挺多，但功能性却很简单，只有两个public方法，即干两件事: <font color=green>invoke BeanFactoryPostProcessor(子类略)和registry BeanPostProcessors to AbstractBeanFactory.beanPostProcessors属性(注册顺序同BeanFactoryPostProcessor调用顺序)</font>。但是这两件事都很重要