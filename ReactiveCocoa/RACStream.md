`RACStream`代表任意流，实际上，`RACStream`是一个抽象类。

其定义了函数式编程中对流的操作符，如`flatMap`、`map`、`mapReduce`、`filter`、`zip`、`concat`等等操作符。

还定义了`name`属性，主要是为了方便我们调试而创建的identifier。

其实现中，为大部分操作符都做了默认的实现，但还有少数操作符作为抽象方法留给其子类实现，如：

```
empty
bind:(RACStreamBindBlock (^)(void))block)
return:(id)value
concat:(RACStream *)stream
zipWith:(RACStream *)stream
```

 实际上，其子类是类簇实现，如empty方法，其实返回的是子类`RACEmptySignal/RACEmptySetSequence`。


 #### map的实现

我们来仔细看看其中一个实现，分析它的特点。这里当然是选择出场率最高的`map`操作符啦～

```objc

- (__kindof RACStream *)map:(id (^)(id value))block {
	NSCParameterAssert(block != nil);

	Class class = self.class;

	return [[self flattenMap:^(id value) {
		return [class return:block(value)];
	}] setNameWithFormat:@"[%@] -map:", self.name];
}

```
