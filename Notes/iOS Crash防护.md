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
String的思路和容器类类似，采取`method swizzling`替换管家方法。

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


