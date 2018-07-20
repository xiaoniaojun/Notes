`RACStream`代表任意流，实际上，`RACStream`是一个抽象类。

其定义了函数式编程中对流的操作符，如`flatMap`、`map`、`mapReduce`、`filter`、`zip`、`concat`等等操作符。

还定义了`name`属性，主要是为了方便我们调试而创建的identifier。

其实现中，为大部分操作符都做了默认的实现，但还有少数操作符作为抽象方法留给其子类实现，如：

```objc
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

学过函数式编程的同学应该知道，这里map内部之所以用flatMap操作符实现，是因为flatMap是Monad，
Monad这种高阶函数是可以模拟出Map（functor）这种低阶函数的。
事实上，用flatMap可以模拟出绝大多数的操作符。

其中，flatMap的实现如下：

```objc
- (__kindof RACStream *)flattenMap:(__kindof RACStream * (^)(id value))block {
	Class class = self.class;

	return [[self bind:^{
		return ^(id value, BOOL *stop) {
			id stream = block(value) ?: [class empty];
			NSCAssert([stream isKindOfClass:RACStream.class], @"Value returned from -flattenMap: is not a stream: %@", stream);

			return stream;
		};
	}] setNameWithFormat:@"[%@] -flattenMap:", self.name];
}
```

首先是拿到self的class，由于RACStream及其子类采用类簇实现，因此采用自省的方式可以动态的查看当前
对象的真实类型。

接下来，调用[self.bind]。
bind方法是`RACStream`定义的抽象方法。

```
///懒式地在接收者中的值上绑定一个block
///
///这个用来在你需要提前终止绑定，或者关闭某些状态
///-flattenMap: 在其他情况下用这个更合适
///block - 一个返回值为RACStreamBindBlock类型的block。每当被绑定的stream被重新求值
///时，这个block就会被调用。这个block不能为nil，也不能返回nil。

///返回一个新的stream，这个stream带代表所有block的懒应用的组合结果。
///
```
bind传入一个block，并要求这个block返回一个`RACStreamBindBlock`类型，
这个类型的定义如下：
`typedef RACStream * _Nullable (^RACStreamBindBlock)(ValueType _Nullable value, BOOL *stop);`

这个block接受一个来自RACStream的value，并且返回一个相同类型的RACStream类型。如果返回值为nil，
代表立即结束。如果设置stop为YES，就会导致在这个bind返回时bind会立即终止。


我们知道flattenMap的作用是，将从订阅源中发送过来的值，传递给我们传入的flattenMap中的block，
作为这个block的参数，然后返回一个新的`RACStream`。
也就是说，falttenMap的作用是**将一个流包装成另一个流**。

在这个实现中，我们为flatten传递的block会被bind函数与接收者进行绑定，每次接收者需要求值的时候，
就会调用我们这个block，生产一个新的流。
其他工作就是检查和设置名字。


注意到bind这个方法是一个抽象方法，在RACStream的子类中才有具体实现。它内部隐含了如何与接收者绑定，
以使得接收者在需要求值的时候调用我们传入的block的逻辑。

### RACSignal#bind:

继续探险，接下来我们就来分析一下`bind:`方法在`RACStream`的直接子类`RACSignal`中的具体实现，

这个方法的体积非常庞大，
我们先概述改方法具体做了哪些事情，再详细的分析。

1. 订阅起源signal的所有值
2. 每当起源signal发送一个值，就用绑定的block进行一次转换
3. 如果绑定的block返回一个signal，则订阅它，并且将它的值全部发送给订阅者，（也就是订阅者接受值）
4. 如果绑定的block要求结束这个bind，则将原始的signal完成(也就是走complete)。
5. 当所有的signal都完成后，则向订阅者发送完成消息。

在这个过程中，如果任何一个signal产生了error，都会被发送给订阅者。

```
- (RACSignal *)bind:(RACSignalBindBlock (^)(void))block {
  // 不允许绑定block为nil
	NSCParameterAssert(block != NULL);

	return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		RACSignalBindBlock bindingBlock = block();

		__block volatile int32_t signalCount = 1;   // indicates self

		RACCompoundDisposable *compoundDisposable = [RACCompoundDisposable compoundDisposable];

		void (^completeSignal)(RACDisposable *) = ^(RACDisposable *finishedDisposable) {
			if (OSAtomicDecrement32Barrier(&signalCount) == 0) {
				[subscriber sendCompleted];
				[compoundDisposable dispose];
			} else {
				[compoundDisposable removeDisposable:finishedDisposable];
			}
		};

		void (^addSignal)(RACSignal *) = ^(RACSignal *signal) {
			OSAtomicIncrement32Barrier(&signalCount);

			RACSerialDisposable *selfDisposable = [[RACSerialDisposable alloc] init];
			[compoundDisposable addDisposable:selfDisposable];

			RACDisposable *disposable = [signal subscribeNext:^(id x) {
				[subscriber sendNext:x];
			} error:^(NSError *error) {
				[compoundDisposable dispose];
				[subscriber sendError:error];
			} completed:^{
				@autoreleasepool {
					completeSignal(selfDisposable);
				}
			}];

			selfDisposable.disposable = disposable;
		};

		@autoreleasepool {
			RACSerialDisposable *selfDisposable = [[RACSerialDisposable alloc] init];
			[compoundDisposable addDisposable:selfDisposable];

			RACDisposable *bindingDisposable = [self subscribeNext:^(id x) {
				// Manually check disposal to handle synchronous errors.
				if (compoundDisposable.disposed) return;

				BOOL stop = NO;
				id signal = bindingBlock(x, &stop);

				@autoreleasepool {
					if (signal != nil) addSignal(signal);
					if (signal == nil || stop) {
						[selfDisposable dispose];
						completeSignal(selfDisposable);
					}
				}
			} error:^(NSError *error) {
				[compoundDisposable dispose];
				[subscriber sendError:error];
			} completed:^{
				@autoreleasepool {
					completeSignal(selfDisposable);
				}
			}];

			selfDisposable.disposable = bindingDisposable;
		}

		return compoundDisposable;
	}] setNameWithFormat:@"[%@] -bind:", self.name];
}

```
