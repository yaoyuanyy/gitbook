# spring(boot)源码阅读笔记
分为三块
- 源码逻辑记录 
- 源码阅读问题池
- spring XXXUtils类的使用例子

## 源码逻辑记录
### aop
```
todo
```

## 源码阅读问题池

### aop 
AspectJProxyFactory干什么用的
实验：
1.定义一个class 用cglib代理debug源码
2.对一个生成的代理对象再次做代理会产生什么现象
$Proxy2内部结构
spring中Advisor与advice的区别


SimpleInstantiationStrategy代码和作用

Enhancer.setCallbackFilter

## spring XXXUtils类的使用例子

### className实例化成Object
```
1. 
Class<?> conditionClass = ClassUtils.resolveClassName(conditionClassName, classloader);
   return (Condition) BeanUtils.instantiateClass(conditionClass);
}

```
