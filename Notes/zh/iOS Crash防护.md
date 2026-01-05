# iOS Crash防护
App Crash是大多数开发者都头疼的事情，尤其是如果线上出现了Crash，排查难度加大不说，及其影响用户体验，甚至可能因此流失客户，因此参考[大白健康系统--iOS APP运行时Crash自动修复系统](https://neyoufan.github.io/2017/01/13/ios/BayMax_HTSafetyGuard/)之后对Crash预防进行了尝试。

## 容器类
容器类指的是比如NSArray、NSMutableArray、NSDictionary等比如越界、插入nil操作造成的Crash

### 方案
通过`method swizzling`替换关键方法，然后在方法中加入防护措施

```Objective-C
Class __NSPlaceholderDictionary = objc_getClass("__NSPlaceholderDictionary");
[self crashProtector_swizzleInstanceMethodWithAClass:__NSPlaceholderDictionary originalSel:@selector(initWithObjects:forKeys:count:) swizzledSel:@selector(crashProtector_initWithObjects:forKeys:count:)];
```

先替换实际调用`initWithObjects:count:`的类的方法，然后实现`crashProtector_initWithObjects:count:`

```Objective-C
- (instancetype)crashProtector_initWithObjects:(id  _Nonnull const [])objects forKeys:(id<NSCopying>  _Nonnull const [])keys count:(NSUInteger)cnt {
    id instance = nil;
    
    @try {
        instance = [self crashProtector_initWithObjects:objects forKeys:keys count:cnt];
    } @catch (NSException *exception) {
        [CrashProtector dealWithException:exception];
        
        NSInteger newCnt = 0;
        id _Nonnull newObject[cnt];
        id<NSCopying>  _Nonnull newKeys[cnt];
        
        for (int i = 0; i < cnt; i++) {
            if (keys[i] && objects[i]) {
                newObject[newCnt] = objects[i];
                newKeys[newCnt] = keys[i];
                newCnt++;
            }
        }
        
        instance = [self crashProtector_initWithObjects:newObject forKeys:newKeys count:newCnt];
    } @finally {
        return instance;
    }
}
```
代码防护了可能出现的初始化插入nil的场景，同时将异常抛给上层进行上报，并返回正常的dictionary。
> Objective-C的容器类大多采用**类簇**的形式，因此需要找到对应的类。

## NSString、NSAttributedString等String类
String的思路和容器类类似，采取`method swizzling`替换关键方法。

### 方案
首先替换相关方法
```Objective-C
Class __NSCFConstantString = objc_getClass("__NSCFString");
[self crashProtector_swizzleInstanceMethodWithAClass:__NSCFConstantString originalSel:@selector(stringByReplacingCharactersInRange:withString:) swizzledSel:@selector(crashProtector_stringByReplacingCharactersInRange:withString:)];
```

同样通过`try-catch`捕获异常抛给上层处理，如果有一场发生返回自己，没有异常则返回对应字符串。

```Objective-C
- (NSString *)crashProtector_stringByReplacingCharactersInRange:(NSRange)range withString:(NSString *)replacement {
    NSString *str = nil;
    
    @try {
        str = [self crashProtector_stringByReplacingCharactersInRange:range withString:replacement];
    } @catch (NSException *exception) {
        [CrashProtector dealWithException:exception];
    } @finally {
        if (str) {
            return str;
        }
        
        return self;
    }
}
```

> 注：具体效果建议和产品经理进行商讨

## NSTimer
NSTimer的问题在于Timer会强引用Target，如果没有在合适的实际Invalidate timer，则会造成Target无法释放，造成内存泄漏，甚至于由于不停的执行IMP，可能会造成Crash。

 ### 方案
 ![未命名文件](https://user-images.githubusercontent.com/22512175/115207981-b01a1900-a12e-11eb-85e0-4c62fd758ef0.png)
 
 我们可以看到，原先的逻辑NSTimer强引用了Target，导致NSTimer不释放，Target就不能释放，基于此，我们可以使用弱引用的方式，加一个中间层，弱引用Target，通过Proxy调用Target的方法，同时持有timer，如果发现Target为空，则`Invalidate timer`。
 
 ![未命名文件 (1)](https://user-images.githubusercontent.com/22512175/115208420-1f900880-a12f-11eb-8fa2-018b7db8cae6.png)

首先`swizzle scheduledTimerWithTimeInterval:target:selector:userInfo:repeats:`
```Objective-C
 Class __NSTimer = object_getClass(NSTimer.class);
    
 [self crashProtector_swizzleInstanceMethodWithAClass:__NSTimer originalSel:@selector(scheduledTimerWithTimeInterval:target:selector:userInfo:repeats:) swizzledSel:@selector(crashProtector_scheduledTimerWithTimeInterval:target:selector:userInfo:repeats:)];
    
+ (NSTimer *)crashProtector_scheduledTimerWithTimeInterval:(NSTimeInterval)ti target:(id)aTarget selector:(SEL)aSelector userInfo:(id)userInfo repeats:(BOOL)yesOrNo {
    NSTimer *timer;
    if (yesOrNo) {
        CrashProtectorProxy *proxy = [CrashProtectorProxy subTargetWithTarget:aTarget selector:aSelector userInfo:userInfo];
        timer = [NSTimer crashProtector_scheduledTimerWithTimeInterval:ti target:proxy selector:@selector(fireProxyTimer) userInfo:userInfo repeats:yesOrNo];
        proxy.timer = timer;
    } else {
        timer = [NSTimer crashProtector_scheduledTimerWithTimeInterval:ti target:aTarget selector:aSelector userInfo:userInfo repeats:yesOrNo];
    }
    
    return timer;
} 
 ```
 
同时代理中调用函数`fireProxyTimer`保证Target被释放了之后停止timer

```Objective-C
- (void)fireProxyTimer {
    if (self.aTarget) {
        if ([self.aTarget respondsToSelector:self.aSelector]) {
            [self.aTarget performSelector:self.aSelector withObject:self.timer];
        }
    } else {
        [self.timer invalidate];
    }
}
```

这样Target便不会受到timer的制约，即便timer没有释放也会跟随自己的生命周期释放，同时释放了时候确保方法不会继续调用。

## unrecognized selector crash
`unrecognized selector crash`Crash大家都熟悉，原因是消息转发机制中走到最后一步也没有找到对应的实现，导致Crash，知道了这一点，我们可以截取其中一个流程，对于没有处理的消息进行容错，同时把Crash报给上层。

### 消息转发机制
1. 从cache里查找IMP，如果找到了就运行对应的函数去执行相应的代码。
2. 如果cache中没有找到就找类的方法列表中是否有对应的方法，查到则缓存。
3. 如果类的方法列表中找不到就到父类的方法列表中查找，一直找到NSObject类为止。
4. 进入消息转发机制
   - 动态方法解析: Method Resolution
   - 快速转发: Fast Rorwarding
   - 完整消息转发: Normal Forwarding
5. 报错

![消息转发流程](https://user-images.githubusercontent.com/22512175/115216191-d643b700-a136-11eb-9089-465686943f6b.png)

可见，即便本类以及父类找不到对应的实现，有3步可供我们进行补救。

### 方案
选择了再第三步进行了拦截，因为在这一步已经确定**消息是否有对象进行处理**，不会影响别的消息流程。

首先还是先`swizzle`

```Objective-C
Class __NSObject = objc_getClass("NSObject");

[self crashProtector_swizzleInstanceMethodWithAClass:__NSObject originalSel:@selector(methodSignatureForSelector:) swizzledSel:@selector(crashProtector_methodSignatureForSelector:)];
[self crashProtector_swizzleInstanceMethodWithAClass:__NSObject originalSel:@selector(forwardInvocation:) swizzledSel:@selector(crashProtector_forwardInvocation:)];
``` 

然后实现自己的处理逻辑

```Objective-C
- (NSMethodSignature *)crashProtector_methodSignatureForSelector:(SEL)aSelector {
    NSMethodSignature *signature = [self crashProtector_methodSignatureForSelector:aSelector];
    if (!signature) {
        signature = [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    
    return signature;
}

- (void)crashProtector_forwardInvocation:(NSInvocation *)anInvocation {
    @try {
        [self crashProtector_forwardInvocation:anInvocation];
    } @catch (NSException *exception) {
        [CrashProtector dealWithException:exception];
    } @finally {

    }
}
```

> 对于最后一步，这边直接`try-catch`是比较偷懒的写法，可以选择将消息转发给代理，这样不会造成Crash，同时创建一个异常抛给上层。

## KVO
KVO存在的问题主要有
* 重复添加观察者（虽然不会crash，但是如果属性修改会通知两次）
* 重复移除观察者crash
* 未移除观察者但是但是被观察者dealloc
* 未实现`-observeValueForKeyPath:ofObject:change:context:`

### 方案
同样可以增加一个中间层来解决这个问题。

![KVO解决方案](https://user-images.githubusercontent.com/22512175/115258303-731c4980-a163-11eb-880d-8dd9c9ef4dd2.png)

这样可以做到add以及remove去重，因为都在Proxy中进行，同时调用`-observeValueForKeyPath:ofObject:change:context:`通知观察者，可以捕获观察者未实现该方法的错误，抛给上层报警。

```Objective-C
Class __NSObject = objc_getClass("NSObject");

[self crashProtector_swizzleInstanceMethodWithAClass:__NSObject originalSel:@selector(addObserver:forKeyPath:options:context:) swizzledSel:@selector(crashProtector_addObserver:forKeyPath:options:context:)];
[self crashProtector_swizzleInstanceMethodWithAClass:__NSObject originalSel:@selector(removeObserver:forKeyPath:) swizzledSel:@selector(crashProtector_removeObserver:forKeyPath:)];
[self crashProtector_swizzleInstanceMethodWithAClass:__NSObject originalSel:NSSelectorFromString(@"dealloc") swizzledSel:@selector(crashProtector_dealloc)];
```

之后实现对应方法，使用代理通过`Dictionary`管理自身的观察者，并做到去重。同时`swizzle dealloc`判断是否有观察者未移除，如果发现则进行移除。

```Objective-C
- (void)crashProtector_addObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options context:(void *)context {
    @try {
        objc_setAssociatedObject(self, crashProtector_KVODefenderKey, crashProtector_KVODefenderValue, OBJC_ASSOCIATION_RETAIN);
        if ([self.kvoProxy addMapInfoWithObserver:observer forKeyPath:keyPath]) {
            [self crashProtector_addObserver:self.kvoProxy forKeyPath:keyPath options:options context:context];
        }
    } @catch (NSException *exception) {
        [CrashProtector dealWithException:exception];
    } @finally {
        
    }
}

- (void)crashProtector_removeObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath {
    @try {
        [self crashProtector_removeObserver:self.kvoProxy forKeyPath:keyPath];
    } @catch (NSException *exception) {
        [CrashProtector dealWithException:exception];
    } @finally {
        if (!isSystemClass(self.class)) {
            [self.kvoProxy removeMapInfoObserver:observer forKeyPath:keyPath];
        }
    }
}

- (void)crashProtector_dealloc {
    if ([(NSString *)objc_getAssociatedObject(self, crashProtector_KVODefenderKey) isEqualToString:crashProtector_KVODefenderValue]) {
        if (self.kvoProxy.infoDic.allKeys.count > 0) {
            for (NSString *keyPath in self.kvoProxy.infoDic.allKeys) {
                [self removeObserver:self.kvoProxy forKeyPath:keyPath];
            }
        }
    }
   [self crashProtector_dealloc];
}
```

实现代理类，添加字典保存观察者以及把方法抛出去，字典的`key`为`keyPath`，`value`为`HashTable`，存储`observer`，同时对于相同`keyPath`进行判断，如果存在相同`observer`，则`return`，返回No，不会再添加，删除时也一样，不过删除没有解决异常，还是交给`try-catch`，也是偷懒的写法，可以解决异常然后自己创造`exception`抛给上层，同时接到通知调用`observer`的`-observeValueForKeyPath:ofObject:change:context:`处理业务逻辑。

```Objective-C
@interface CrashProtector_KVOProxy : NSObject

@property (nonatomic, strong) NSMutableDictionary *infoDic;

@end

@implementation CrashProtector_KVOProxy

- (instancetype)init {
    if (self = [super init]) {
        self.infoDic = NSMutableDictionary.new;
    }
    
    return self;
}

- (BOOL)addMapInfoWithObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath {
    @synchronized (self) {
        if (keyPath.length == 0) {
            // 返回YES走try catch，把错误抛出去
            return YES;
        }
        if ([self.infoDic.allKeys containsObject:keyPath]) {
            if ([self.infoDic[keyPath] containsObject:observer]) {
                return NO;
            }
            
            NSHashTable *table = self.infoDic[keyPath];
            if (table.count > 0) {
                if (![table containsObject:observer]) {
                    [table addObject:observer];
                    return NO;
                }
            }
        }
        
        
        NSHashTable *table = [[NSHashTable alloc] initWithOptions:NSPointerFunctionsWeakMemory capacity:3];
        [table addObject:observer];
        self.infoDic[keyPath] = table;
        
        return YES;
    }
}

- (void)removeMapInfoObserver:(NSObject *)observer forKeyPath:(NSString *)keyPath {
    @synchronized (self) {
        NSHashTable *table = self.infoDic[keyPath];
        if (table) {
            [table removeObject:observer];
            if (table.count == 0) {
                [self.infoDic removeObjectForKey:keyPath];
            }
        }
    }
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    NSHashTable *info = self.infoDic[keyPath];
    for (NSObject *object in info) {
        @try {
            if ([object respondsToSelector:@selector(observeValueForKeyPath:ofObject:change:context:)]) {
                [object observeValueForKeyPath:keyPath ofObject:object change:change context:context];
            }
        } @catch (NSException *exception) {
            [CrashProtector dealWithException:exception];
        } @finally {
            
        }
    }
}
```

 > 相关操作注意线程安全
 
 ## 野指针
 野指针的定义是如果指针指向的内存地址已被回收，但是该指针仍然指向这个地址。
 
 野指针的问题一般较难调试，因为不好复现并且提供的日志信息比较有限。Xcode提供了一种定位野指针的方法-**Zombie Object**
 
 ![Zombie Object](https://user-images.githubusercontent.com/22512175/115373102-2043a000-a1fe-11eb-8fa3-9f54108b9de1.png)

这就给我们提供了思路，在线上提供类似于`Zombie`的机制，可以补货异常并不造成崩溃。

### 方案
`swizzle dealloc`，将对象的`isa`指向动态生成的**僵尸类**，然后可以通过截取字符串获取原始类名（因为僵尸类是动态生成的，即为原始类名加上`_CrashProtecotrZoombie_`前缀），最后一步就是前面消息的拦截，阻止`Crash`，然后设定僵尸类的存储空间，防止太多类没释放触发`OOM`。
 
```Objective-C
- (void)handleDeallocObject:(__unsafe_unretained id)object {
    // 指向动态生成的类，用 _CrashProtecotrZoombie_ 拼接原有类名
    const char *clsName = class_getName(object_getClass(object));
    const char *zoombizPrx = "_CrashProtecotrZoombie_";
    char buff[1024];
    const char *zombieClassName = strcat(strcpy(buff, zoombizPrx), clsName);

    Class zombieCls = objc_lookUpClass(zombieClassName);
    if(zombieCls) return;
    zombieCls = objc_allocateClassPair([NSObject class], zombieClassName, 0);;
    
    objc_registerClassPair(zombieCls);
    
    class_addMethod([zombieCls class], @selector(forwardingTargetForSelector:), (IMP)forwardingTargetForSelector, "v@:@");
    
    object_setClass(object, zombieCls);
    if (_zombieList.count < 10) {
        [_zombieList addObject:object];
    } else {
        id _object = _zombieList.firstObject;
        [_zombieList removeObject:_object];
        if ([_object respondsToSelector:@selector(crashProtector_dealloc)]) {
            [_object performSelector:@selector(crashProtector_dealloc)];
        }
        
        [_zombieList addObject:object];
    }
}

void forwardingTargetForSelector(id object, SEL _cmd, SEL aSelector) {
    NSString *className = NSStringFromClass([object class]);
    NSString *realClass = [className stringByReplacingOccurrencesOfString:@"_CrashProtecotrZoombie_" withString:@""];

    NSLog(@"[%@ %@] message sent to deallocated instance %@", realClass, NSStringFromSelector(aSelector), object);
}
```

[DEMO](https://github.com/METISU/CrashProtector)
