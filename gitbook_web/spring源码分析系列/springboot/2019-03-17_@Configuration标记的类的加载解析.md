# springboot @Configuration标记的类的加载解析
## @Configuration作用
```
/**
 * 一个类：声明一个或多个@Bean mothod，并且被spring容器处理来生成bean definitions，同时在运行时为这些beans提供服务(instantiate, configure and return bean)。
 * 例如：
 * @Configuration
 * public class AppConfig {
 *     @Bean
 *     public MyBean myBean() {
 *         // instantiate, configure and return bean ...
 *     }
 * }
 * 创建一个@Configuration class时的约束如下：
 * - Configuration classes must be provided as classes (即不是从工厂方法返回的实例)，允许通过生成的子类进行运行时增强
 * - Configuration classes must be non-final.
 * - Configuration classes must be non-local (i.e. may not be declared within a method).
 * - Any nested configuration classes must be declared as static 
 * - @Bean methods may not in turn create further configuration classes (any such instances will be treated as regular beans, with their configuration annotations remaining undetected).
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {
    String value() default "";
}
```

## @Configuration标记的类的加载源码分析

入口
ConfigurationClassPostProcessor.processConfigBeanDefinitions(BeanDefinitionRegistry registry).此方法为获取所有的@Configuration class集合用于reader解析
```

public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    ... 
    // 读取和解析app下所有class文件
	parser.parse(candidates);
	parser.validate();

	Set<ConfigurationClass> configClasses = new LinkedHashSet<ConfigurationClass>(parser.getConfigurationClasses());
    ... 

	this.reader = new ConfigurationClassBeanDefinitionReader(
		registry, this.sourceExtractor, this.resourceLoader, this.environment,
		this.importBeanNameGenerator, parser.getImportRegistry());

    //核心方法，configClasses为@Configuration class集合,使用reader读取这些class集合
this.reader.loadBeanDefinitions(configClasses);
    ... 
}
```
this.reader.loadBeanDefinitions(configClasses)实际上是遍历@Configuration class集合，挨个解析加载。因为@Configuration可以结合一下注解@Import, @ImportSource, @PropertySource, @Profile @ComponentScan, nested @Configuration class, @EnableXXX 实现功能，所以针对这些加载时都要解析。看loadBeanDefinitions方法，如下
```
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
	TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
	for (ConfigurationClass configClass : configurationModel) {
		loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
	}
}
```
遍历解析，具体的解析都在loadBeanDefinitionsForConfigurationClass方法。根据传入的Configuration class，开始解析
```
private void loadBeanDefinitionsForConfigurationClass(ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {


    // 依据@ConditionalXXX的结果判断importedBy属性
    if(trackedConditionEvaluator.shouldSkip(configClass)) {
		String beanName = configClass.getBeanName();
		if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
			this.registry.removeBeanDefinition(beanName);
		}
		this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
		return;
	}

	if (configClass.isImported()) {
		registerBeanDefinitionForImportedConfigurationClass(configClass);
	}
	for (BeanMethod beanMethod : configClass.getBeanMethods()) {
		loadBeanDefinitionsForBeanMethod(beanMethod);
	}
	loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
	loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```
可以看到，一个@Configuration class的解析过程就在这个方法里。在往下走就是每个@Configuration class自己的解析逻辑了。