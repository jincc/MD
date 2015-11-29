###Opetation
###catch ,catchTo
`- (RACSignal *)catch:(RACSignal * (^)(NSError *error))catchBlock`捕获error，当原signal sendError的时候，return的errorSignal subscriber
<pre><code>	return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		RACSerialDisposable *catchDisposable = [[RACSerialDisposable alloc] init];

		RACDisposable *subscriptionDisposable = [self subscribeNext:^(id x) {
			[subscriber sendNext:x];
		} error:^(NSError *error) {
			RACSignal *signal = catchBlock(error);
			------------------------------------
			NSCAssert(signal != nil, @"Expected non-nil signal from catch block on %@", self);
			catchDisposable.disposable = [signal subscribe:subscriber];
			-------------------------------------
		} completed:^{
			[subscriber sendCompleted];
		}];

		return [RACDisposable disposableWithBlock:^{
			[catchDisposable dispose];
			[subscriptionDisposable dispose];
		}];
	}] setNameWithFormat:@"[%@] -catch:", self.name];</code></pre>

###try ,tryMap
	`- (RACSignal *)try:(BOOL (^)(id value, NSError **errorPtr))tryBlock`执行tryBlock在每一次发送值的时候，当block返回true sendNext ,else endError
	<pre><code>
	return [[self flattenMap:^(id value) {
		NSError *error = nil;
		BOOL passed = tryBlock(value, &error);
		return (passed ? [RACSignal return:value] : [RACSignal error:error]);
		----------------------------------------
	}] setNameWithFormat:@"[%@] -try:", self.name];</code></pre>
	
###defer
`+ (RACSignal *)defer:(RACSignal * (^)(void))block;`延迟创建signal，当有订阅者的时候才创建.
<pre><code>	return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		return [block() subscribe:subscriber];
		---------------------
	}] setNameWithFormat:@"+defer:"];</code></pre>

###initially
`- (RACSignal *)initially:(void (^)(void))block`执行block，当有订阅者的时候
<pre><code>return [[RACSignal defer:^{
		block();
		return self;
	}] setNameWithFormat:@"[%@] -initially:", self.name];</code></pre>
	
###finally
`- (RACSignal *)finally:(void (^)(void))block`执行block，when doError|doCompleted
<pre><code>	return [[[self
		doError:^(NSError *error) {
			block();
		}]
		doCompleted:^{
			block();
		}]
		setNameWithFormat:@"[%@] -finally:", self.name];</code></pre>

###buffer
		
`- (RACSignal *)bufferWithTime:(NSTimeInterval)interval onScheduler:(RACScheduler *)scheduler`每一次在interval 时间内，只要有Next发送就缓存他们的值(Nexts),时间过后，返回RACTuple（nexts）
<pre><code>return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		RACSerialDisposable *timerDisposable = [[RACSerialDisposable alloc] init];
		NSMutableArray *values = [NSMutableArray array];

		void (^flushValues)() = ^{
			@synchronized (values) {
				[timerDisposable.disposable dispose];

				if (values.count == 0) return;

				RACTuple *tuple = [RACTuple tupleWithObjectsFromArray:values];
				//当bufferTime时间带了之后，释放缓存池里面的内容，发送一个tuple对象给订阅者,里面的值为time期间缓存的Nexts
				[values removeAllObjects];
				[subscriber sendNext:tuple];
			}
		};

		RACDisposable *selfDisposable = [self subscribeNext:^(id x) {
			@synchronized (values) {
				if (values.count == 0) {
				//当signal每一次在Time期间，values缓存Nexts，scheduler延时调用flushValues（）
					timerDisposable.disposable = [scheduler afterDelay:interval schedule:flushValues];
				}

				[values addObject:x ?: RACTupleNil.tupleNil];
			}
		} error:^(NSError *error) {
			[subscriber sendError:error];
		} completed:^{
			flushValues();
			[subscriber sendCompleted];
		}];

		return [RACDisposable disposableWithBlock:^{
			[selfDisposable dispose];
			[timerDisposable dispose];
		}];
	}] setNameWithFormat:@"[%@] -bufferWithTime: %f onScheduler: %@", self.name, (double)interval, scheduler];
</code></pre>
###collect
`- (RACSignal *)collect;`

Collects all receiver's `next`s into a NSArray. Nil values will be converted to NSNull.
###takeLast
###combineLatestWith
<pre><code>return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];

		__block id lastSelfValue = nil;
		__block BOOL selfCompleted = NO;

		__block id lastOtherValue = nil;
		__block BOOL otherCompleted = NO;

		void (^sendNext)(void) = ^{
			@synchronized (disposable) {
				if (lastSelfValue == nil || lastOtherValue == nil) return;
				[subscriber sendNext:RACTuplePack(lastSelfValue, lastOtherValue)];
				--------------------------
				//当两个信号都Next得时候，sendNext
			}
		};

		RACDisposable *selfDisposable = [self subscribeNext:^(id x) {
			@synchronized (disposable) {
				lastSelfValue = x ?: RACTupleNil.tupleNil;
				sendNext();
			}
		} error:^(NSError *error) {
			[subscriber sendError:error];
		} completed:^{
			@synchronized (disposable) {
				selfCompleted = YES;
				if (otherCompleted) [subscriber sendCompleted];
				--------------------------
				//当两个信号都sendcompleted的时候，signal才sendcompleted
			}
		}];

		[disposable addDisposable:selfDisposable];

		RACDisposable *otherDisposable = [signal subscribeNext:^(id x) {
			@synchronized (disposable) {
				lastOtherValue = x ?: RACTupleNil.tupleNil;
				sendNext();
			}
		} error:^(NSError *error) {
			[subscriber sendError:error];
			---------------------
			//当其中任意一个信号sendError的时候，signal sendError
		} completed:^{
			@synchronized (disposable) {
				otherCompleted = YES;
				if (selfCompleted) [subscriber sendCompleted];
			}
		}];

		[disposable addDisposable:otherDisposable];

		return disposable;
	}] setNameWithFormat:@"[%@] -combineLatestWith: %@", self.name, signal];
</code></pre>
###merge
合并所有signal的Nexts,可以理解为或(|),只要有一个signal sendNext之后，转发该值
<pre><code>+ (RACSignal *)merge:(id<NSFastEnumeration>)signals {
	NSMutableArray *copiedSignals = [[NSMutableArray alloc] init];
	for (RACSignal *signal in signals) {
		[copiedSignals addObject:signal];
	}

	return [[[RACSignal
		createSignal:^ RACDisposable * (id<RACSubscriber> subscriber) {
			for (RACSignal *signal in copiedSignals) {
			//依次转发signal
				[subscriber sendNext:signal];
			}

			[subscriber sendCompleted];
			return nil;
		}]
		flatten]
		//最后放平
		setNameWithFormat:@"+merge: %@", copiedSignals];
}
</code></pre>
###concat 前面信号完成之后，订阅下一个信号
<pre><code>- (RACSignal *)concat:(RACSignal *)signal {
	return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		RACSerialDisposable *serialDisposable = [[RACSerialDisposable alloc] init];

		RACDisposable *sourceDisposable = [self subscribeNext:^(id x) {
			[subscriber sendNext:x];
		} error:^(NSError *error) {
			[subscriber sendError:error];
		} completed:^{
			RACDisposable *concattedDisposable = [signal subscribe:subscriber];
			-------------------------------
			serialDisposable.disposable = concattedDisposable;
		}];

		serialDisposable.disposable = sourceDisposable;
		return serialDisposable;
	}] setNameWithFormat:@"[%@] -concat: %@", self.name, signal];
}</code></pre>
###then
ignoreValues + concat
###- (RACDisposable *)setKeyPath:(NSString *)keyPath onObject:(NSObject *)object nilValue:(id)nilValue 
绑定next值到object.keypath
###interval
定时器
###takeUntil
`- (RACSignal *)takeUntil:(RACSignal *)signalTrigger`获取Nexts直到signalTrigger发送next|complement
<pre><code>- (RACSignal *)takeUntil:(RACSignal *)signalTrigger {
	return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];
		void (^triggerCompletion)(void) = ^{
			[disposable dispose];
			[subscriber sendCompleted];
		};

		RACDisposable *triggerDisposable = [signalTrigger subscribeNext:^(id _) {
			triggerCompletion();
		} completed:^{
			triggerCompletion();
			
		}];</code></pre>

###takeUntilReplacement获取Nexts，直到replacement发送时间Next,error,complemt，然后转发replacement的值
<pre><code>
- (RACSignal *)takeUntilReplacement:(RACSignal *)replacement {
	return [RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		RACSerialDisposable *selfDisposable = [[RACSerialDisposable alloc] init];

		RACDisposable *replacementDisposable = [replacement subscribeNext:^(id x) {
			[selfDisposable dispose];
			[subscriber sendNext:x];
		} error:^(NSError *error) {
			[selfDisposable dispose];
			[subscriber sendError:error];
		} completed:^{
			[selfDisposable dispose];
			[subscriber sendCompleted];
		}];</code></pre>
		
###switchToLatest
###+ (RACSignal *)switch:(RACSignal *)signal cases:(NSDictionary *)cases default:(RACSignal *)defaultSignal
通过signal发送的Next的值，去匹配cases里面对应的signal

<pre><code>    RACSubject *suject = [RACSubject subject];
    [[RACSignal switch:suject
                cases:@{@"one":[RACSignal return:@"one"]}
                default:[RACSignal empty]]subscribeNext:^(id x) {
        NSLog(@"%@",x); //one
    }];
    [suject sendNext:@"one"];</code></pre>

###+(RACSignal *)if:(RACSignal *)boolSignal then:(RACSignal *)trueSignal else:(RACSignal *)falseSignal 

<pre><code>    RACSubject *suject = [RACSubject subject];
    [[RACSignal if:suject then:[RACSignal return:@(true)] else:[RACSignal return:@(false)]]subscribeNext:^(id x) {
        NSLog(@"%@",x);//true
    }];
    [suject sendNext:@(true)];</code></pre>
###first  ,toArray
<pre><code>    RACSignal *s = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        
        [subscriber sendNext:@1];
        [subscriber sendNext:@2];
        [subscriber sendError:nil];
        return nil;
    }];
    NSLog(@"----%@",[s  first]);//1
    NSLog(@"----%@",[s  toArray]);//1,2</code></pre>
###replay 
创建connection ，立即connect
<pre><code>- (RACSignal *)replay {
	RACReplaySubject *subject = [[RACReplaySubject subject] setNameWithFormat:@"[%@] -replay", self.name];

	RACMulticastConnection *connection = [self multicast:subject];
	[connection connect];

	return connection.signal;
}</code></pre>

###replayLast

<pre><code>- (RACSignal *)replayLast {
//这里创建了一个replaysuject，他只能存贮最后一个Next，所以replayLast创建的signal只会转发最后一个Next的值给所有的订阅者们
	RACReplaySubject *subject = [[RACReplaySubject replaySubjectWithCapacity:1] setNameWithFormat:@"[%@] -replayLast", self.name];

	RACMulticastConnection *connection = [self multicast:subject];
	[connection connect];

	return connection.signal;
}</code></pre>

###replayLazily
仅仅当第一个订阅者订阅的时候，才connect出发副作用
<pre><code>- (RACSignal *)replayLazily {
	RACMulticastConnection *connection = [self multicast:[RACReplaySubject subject]];
	return [[RACSignal
		defer:^{
		---------------------
			[connection connect];
			return connection.signal;
		}]
		setNameWithFormat:@"[%@] -replayLazily", self.name];
}</code></pre>
###- (RACSignal *)timeout:(NSTimeInterval)interval onScheduler:(RACScheduler *)scheduler
源信号在time期间未完成，返回一个timeout的signal

###deliverOn
添加副作用到给定的调度器

###- (RACSignal *)any:(BOOL (^)(id object))predicateBlock 
只要有一个next通过了predicateBlock，send true

###- (RACSignal *)all:(BOOL (^)(id object))predicateBlock 
所有的Next的值通过predicateBlock，send true
###materialize map(event)