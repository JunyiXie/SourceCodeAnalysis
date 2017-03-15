# SourceCodeAnalysis
Source code analysis



# Aspects 

## 简介

Aspects是一个面向切面编程的库。
如果想深入了解iOS Runtime中的消息发送机制，Aspects的源码是值得分析的。    

**项目主页**
[Aspects](https://github.com/steipete/Aspects)


## 整体分析

阅读Aspects的源码需要以下知识作为基础
1. Objective-C Runtime
2. 理解OC的消息分发机制
3. KVO中的指针交换技术

## 核心实现
Aspects的核心实现就是利用Runtime中的消息分发机制如图：

![原理](Aspects-源码分析/原理.png)

**Aspects通过把selector的方法替换为msg_forward方法转发 转而调用 forwardInvocation（forwardInvocation的实现被Aspects替换，将原来的方法实现与添加的实现组合在了一起）**



## 核心源码分析
这是Aspects 面向切面编程的入口方法
```objectivec
- (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error {
    return aspect_add(self, selector, options, block, error);
}
```


这段代码可以分三部分来看
1. **aspect_isSelectorAllowedAndTrack 这个方法 对父子类同时hook一个方法进行了一些限制**
2. **aspect_getContainerForObject 通过Runtime添加关联值的方式 管理hook的方法**
3. **aspect_prepareClassAndHookSelector 这是核心的实现，涉及到动态生成子类，改变isa指针的指向，改变方法的实现 一系列操作**
```objectivec
static id aspect_add(id self, SEL selector, AspectOptions options, id block, NSError **error) {
    NSCParameterAssert(self);
    NSCParameterAssert(selector);
    NSCParameterAssert(block);

    __block AspectIdentifier *identifier = nil;
    aspect_performLocked(^{
        if (aspect_isSelectorAllowedAndTrack(self, selector, options, error)) {
            //一个实例 只有一个container
            //这是区分实例对象和类对象的关键
            //实例对象可以有很多个，但是同一个类的类对象只能有一个
            AspectsContainer *aspectContainer = aspect_getContainerForObject(self, selector);
            //原来的selector block
            identifier = [AspectIdentifier identifierWithSelector:selector object:self options:options block:block error:error];
            if (identifier) {
                //container 里 存有 identifier (selector,block)
                [aspectContainer addAspect:identifier withOptions:options];

                // Modify the class to allow message interception.
                aspect_prepareClassAndHookSelector(self, selector, error);
            }
        }
    });
    return identifier;
}


```




### 核心方法
 **aspect_prepareClassAndHookSelector这是核心的实现，涉及到动态生成子类，改变isa指针，改变方法的实现 一系列操作**

```objectivec
static void aspect_prepareClassAndHookSelector(NSObject *self, SEL selector, NSError **error) {
    NSCParameterAssert(selector);
    //动态创建子类，改变forwardInvocation方法的实现
    Class klass = aspect_hookClass(self, error);
    Method targetMethod = class_getInstanceMethod(klass, selector);
    IMP targetMethodIMP = method_getImplementation(targetMethod);

    if (!aspect_isMsgForwardIMP(targetMethodIMP)) {
        // Make a method alias for the existing method implementation, it not already copied.
        const char *typeEncoding = method_getTypeEncoding(targetMethod);
        SEL aliasSelector = aspect_aliasForSelector(selector);
        if (![klass instancesRespondToSelector:aliasSelector]) {
            //子类的aliasSelector的实现为 当前类的selector
            __unused BOOL addedAlias = class_addMethod(klass, aliasSelector, method_getImplementation(targetMethod), typeEncoding);
            NSCAssert(addedAlias, @"Original implementation for %@ is already copied to %@ on %@", NSStringFromSelector(selector), NSStringFromSelector(aliasSelector), klass);
        }

        //selector方法替换为_objc_msgForward
        // We use forwardInvocation to hook in.
        class_replaceMethod(klass, selector, aspect_getMsgForwardIMP(self, selector), typeEncoding);
        AspectLog(@"Aspects: Installed hook for -[%@ %@].", klass, NSStringFromSelector(selector));
    }
}
```


### 动态生成子类，改变isa指针
```objectivec
#pragma mark - Hook Class

static Class aspect_hookClass(NSObject *self, NSError **error) {
    NSCParameterAssert(self);
    
    //[self class] KVO可能改变了isa指针的指向
    Class statedClass = self.class;
    
    // object_getClass 能准确的找到isa指针
    Class baseClass = object_getClass(self);
    NSString *className = NSStringFromClass(baseClass);

    // Already subclassed
    //如果已经子类化了 就返回
    if ([className hasSuffix:AspectsSubclassSuffix]) {
        return baseClass;

        //如果是类 就改掉类的forwardInvocation 而不是一个子类对象
        // We swizzle a class object, not a single object.
    }else if (class_isMetaClass(baseClass)) {
        return aspect_swizzleClassInPlace((Class)self);
        
        //考虑到KVO,KVO的底层实现,交换了isa指针
        // Probably a KVO'ed class. Swizzle in place. Also swizzle meta classes in place.
    }else if (statedClass != baseClass) {
        return aspect_swizzleClassInPlace(baseClass);
    }

    // Default case. Create dynamic subclass.
    const char *subclassName = [className stringByAppendingString:AspectsSubclassSuffix].UTF8String;
    Class subclass = objc_getClass(subclassName);

    if (subclass == nil) {
        
        // 通过创建新子类的方式
        subclass = objc_allocateClassPair(baseClass, subclassName, 0);
        if (subclass == nil) {
            NSString *errrorDesc = [NSString stringWithFormat:@"objc_allocateClassPair failed to allocate class %s.", subclassName];
            AspectError(AspectErrorFailedToAllocateClassPair, errrorDesc);
            return nil;
        }
        // forwardInvocation 替换成 (IMP)_ASPECTS_ARE_BEING_CALLED__
        aspect_swizzleForwardInvocation(subclass);

        
        //子类的class方法返回当前被hook的对象的class
        aspect_hookedGetClass(subclass, statedClass);
        aspect_hookedGetClass(object_getClass(subclass), statedClass);
        
        objc_registerClassPair(subclass);
    }

    //将当前self设置为子类，这里其实只是更改了self的isa指针而已, 这里hook了子类的forwardInvocation方法，再次使用当前类时，其实是使用了子类的forwardInvocation方法。
    object_setClass(self, subclass);
    return subclass;
}
```