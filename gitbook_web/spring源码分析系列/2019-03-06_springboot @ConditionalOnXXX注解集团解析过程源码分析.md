# springboot @ConditionalOnXXX注解集团解析过程源码分析
- 入口：当@ConditionalOnXXX作用于带有@Configuration的类或带有@Bean的方法时，解析入口为ConfigurationClassParser.processConfigurationClass(ConfigurationClass)方法

```
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
    // 获取@Conditional注解，检验condition条件是否匹配。匹配则通过，往下执行；否则return
	if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
		return;
	}

	ConfigurationClass existingClass = this.configurationClasses.get(configClass);
	
	......
	
	// Recursively(递归) process the configuration class and its superclass hierarchy.
	SourceClass sourceClass = asSourceClass(configClass);
	do {
	// 接着处理sourceClass的any member (nested) classes、 @PropertySource、@ComponentScan
	// @Import、@ImportResource、@Bean、Process superclass
		sourceClass = doProcessConfigurationClass(configClass, sourceClass);
	}
	while (sourceClass != null);

	this.configurationClasses.put(configClass, configClass);
}
```
	
- 深入分析
	conditionEvaluator.shouldSkip()作为condition判断的总入口，此方法传入Metadata和ConfigurationPhase，方法代码只列出关键逻辑

```
public boolean shouldSkip(AnnotatedTypeMetadata metadata, ConfigurationPhase phase) {
	if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
		return false;
	} 
	
   // 从metadata拿到conditions(包含@ConditionalOnXXX的集合)
	for (Condition condition : conditions) {
		if (requiredPhase == null || requiredPhase == phase) {
		// 关键代码，具体的匹配逻辑
			if (!condition.matches(this.context, metadata)) {
				return true;
			}
		}
	}

	return false;
}
```
每个@ConditionalOnXXX的匹配结果通过Condition.matches()方法得出结果，具体的匹配逻辑就在这个方法里。可以看出：只要有一个Condition不匹配，方法直接返回true。这里的Condition实际上是SpringbootCondition,而SpringbootCondition是OnXXXCondition的父类，这是一个模板方法模式。注意一点：@ConditionalOnXXX对应OnXXXCondition类，方法内部如下

```
public final boolean matches(ConditionContext context,AnnotatedTypeMetadata metadata) {
    String classOrMethodName = getClassOrMethodName(metadata);
    // 每个OnXXXCondition类调用自己的getMatchOutcome()方法，outcome封装了匹配结果
	ConditionOutcome outcome = getMatchOutcome(context, metadata);
	// 匹配结果放入ConditionEvaluationReport中，用于debug日志输出
	recordEvaluation(context, classOrMethodName, outcome);
	return outcome.isMatch();
}
```

到这里都是通用的匹配逻辑，具体的OnXXXCondition自己的属性，字段匹配都有自己的getMatchOutcome()方法负责，这里以@ConditionalOnProperty(prefix = "spring.aop", name = "auto", havingValue = "true", matchIfMissing = true)为例说下OnPropertyCondition.getMatchOutcome()的逻辑，如下
```
public ConditionOutcome getMatchOutcome(ConditionContext context,
		AnnotatedTypeMetadata metadata) {
	List<AnnotationAttributes> allAnnotationAttributes = annotationAttributesFromMultiValueMap(
			metadata.getAllAnnotationAttributes(ConditionalOnProperty.class.getName()));
	for (AnnotationAttributes annotationAttributes : allAnnotationAttributes) {
		ConditionOutcome outcome = determineOutcome(annotationAttributes,context.getEnvironment());
		(outcome.isMatch() ? match : noMatch).add(outcome.getConditionMessage());
	}
	if (!noMatch.isEmpty()) {
		return ConditionOutcome.noMatch(ConditionMessage.of(noMatch));
	}
	return ConditionOutcome.match(ConditionMessage.of(match));
}
	
private ConditionOutcome determineOutcome(AnnotationAttributes annotationAttributes,
		PropertyResolver resolver) {
	Spec spec = new Spec(annotationAttributes);
	spec.collectProperties(resolver, missingProperties, nonMatchingProperties);
}
    
private void collectProperties(PropertyResolver resolver, List<String> missing,List<String> nonMatching) {
		if (this.relaxedNames) {
			resolver = new RelaxedPropertyResolver(resolver, this.prefix);
		}
		for (String name : this.names) {
			String key = (this.relaxedNames ? name : this.prefix + name);
			if (resolver.containsProperty(key)) {
				if (!isMatch(resolver.getProperty(key), this.havingValue)) {
					nonMatching.add(name);
				}
			}
			else {
				if (!this.matchIfMissing) {
					missing.add(name);
				}
			}
	}
}
		
```
resolver是用来解析Environment中配置文件的，所以collectProperties()方法的逻辑为：如果resolver在配置文件中找到了spring.aop.auto这个key且spring.aop.auto=havingValue则匹配成功，或者配置文件中没有spring.aop.auto这个key且matchIfMissing为true则匹配成功，其他任何逻辑都是匹配失败。

- 看到这里，是不是对我们总迷惑的havingValue和matchIfMissing的含义彻底的明白了。所以，走源码可以为你解惑