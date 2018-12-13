KVO（Key-Value Observing）是 Objective-C 中对于观察者模式的一种实现，它可以让你监听任意对象中属性的改变。使用时通过：

```objective-c
[obj addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:nil];
```

来观察一个对象`obj`中的`name`属性，当属性发生改变的时候，则会调用`self`中的：

```objective-c
- (void)observeValueForKeyPath:(NSString *)keyPath 
    				  ofObject:(id)object
                        change:(NSDictionary *)change 
                       context:(void *)context {
    if ([keyPath isEqualToString:@"name"]) {
        // ...
    }
}
```

被观察的对象也可以**手动触发**自己特定属性的 KVO，只需要：

```objective-c
// 在被观察的对象中
[self willChangeValueForKey:@"name"];
// ...这里可以修改name的值
[self didChangeValueForKey:@"name"]; // 发生KVO通知
```

## 原理

KVO是通过OC的Runtime来实现的，在第一次调用`addObserver`为一个对象添加 KVO 的时候，会为该对象的类型<u>动态创建</u>一个子类型，然后将该对象的`isa`指针指向这个新的类型，这个可以通过打印对象的`isa`指针得知：

```shell
# 在addObserver之前
(lldb) p self.obj->isa
(Class) $0 = KVOObject
# 在addObserver之后
(lldb) p self.obj->isa
(Class) $1 = NSKVONotifying_KVOObject
```

这个类型会<u>为被观察的属性重载`setter`方法</u>，当方法被调用的时候进行真正的通知，如果通过`[obj methodForSelector:@selector(setName:)]`打印它的`setName:`方法会发现已经变成了另一个方法。

子类的`class`方法也会被重载，返回它父类的`Class`，让你认为 obj 还是原本的类型，从而达到隐藏动态生成的子类型的目的。

## 实现

实现一个简单的KVO的Demo，观察`id`类型的属性，并在属性发生改变的时候使用`NSNotification`来发送一个通知，实现具体分为以下几步：

1. **为被观察的对象动态创建一个子类**

   首先定义一个方法`createSubclassFor:`来为一个指定对象创建一个子类型，动态创建的子类继承自对象原本的类型：

   ```objective-c
   - (Class)createSubclassFor:(NSObject *)object {
       NSString *newClassname = [NSString stringWithFormat:@"CustomKVONotifying_%@", [object class]];
       Class newClass = NSClassFromString(newClassname);
       if (newClass) {
           return newClass;
       }
       
       // 1. 创建一个继承自原本类型的新类型
       Class origClass = object_getClass(object);
       newClass = objc_allocateClassPair(origClass, [newClassname cStringUsingEncoding:NSUTF8StringEncoding], 0);
       // 2. 添加一个class方法
       Method origMethod = class_getInstanceMethod([object class], @selector(class));
       class_addMethod(newClass, @selector(class), (IMP)customKVOClass, method_getTypeEncoding(origMethod));
       // 3. 注册类型
       objc_registerClassPair(newClass);
       
       return newClass;
   }
   ```

2. **实现class方法**

   上面创建类型的时候添加了一个`class`方法，它的`IMP`是一个C函数：

   ```objective-c
   Class customKVOClass(id self, SEL sel) {
       return class_getSuperclass(object_getClass(self));
   }
   ```

   <u>返回`self`的父类的类型</u>，这是为了对上层隐藏派生类的存在。

3. **替换原始对象的isa指针**

   ```objective-c
   Class newClass = [self createSubclassFor:object];
       
   // 1. 获取object对象isa指针实际指向的类型
   Class cls = object_getClass(object);
   // 2. 如果object不是KVO类型，重新设置它的isa指针
   if (![NSStringFromClass(cls) hasPrefix:@"CustomKVONotifying"]) {
       object_setClass(object, newClass);
   }
   
   // 被观察的属性的setter方法
   NSString *setter = [NSString stringWithFormat:@"set%@", [keyPath capitalizedString]];
   SEL setterSel = NSSelectorFromString([NSString stringWithFormat:@"%@:", setter]);
   Method setterMethod = class_getInstanceMethod([object class], setterSel);
   
   // 3. 添加被观察属性的setter方法
   class_addMethod(newClass, setterSel, (IMP)setObjectAndNotify, method_getTypeEncoding(setterMethod));
   ```

    通过`object_setClass`修改对象的`isa`指针，然后为被观察的属性重载`setter`方法，注意在DEMO中默认属性都是`id`类型的，<u>实际上系统的KVO为不同类型的属性实现了不同的`setter`</u>；

   > `class`方法可能被重载拿不到真正的类型，`object_getClass`函数会直接读取对象的`isa`指针，拿到对象真正的类型

4. **实现Setter函数**

   自定义的Setter函数的作用在于：1. 通知观察者属性发生改变；2. 调用`super`的setter方法设置属性值。

   由于`setObjectAndNotify`是一个C函数，我们只能通过`objc_msgSendSuper`这个函数来向它的父类方法发送消息，以达到调用`super`方法的目的，`objc_msgSendSuper`的方法签名如下：

   ```objective-c
   id objc_msgSendSuper(struct objc_super *super, SEL cmd, ...);
   ```

   通过 runtime 的源码可知`objc_super`结构体的定义如下：

   ```objective-c
   struct objc_super {
       __unsafe_unretained id receiver; // 需要调用super的对象
       __unsafe_unretained Class super_class; // 对象父类的类型
   };
   ```

   这里我们需要自己构造一个`objc_super`并调用`objc_msgSendSuper`来调用父类的方法修改属性值：

   ```objective-c
   #import <objc/message.h>
   
   void setObjectAndNotify(id self, SEL sel, id obj) {
       struct objc_super superStruct = {
           .receiver = self,
           .super_class = class_getSuperclass(object_getClass(self))
       };
       ((id(*)(struct objc_super *, SEL, ...))objc_msgSendSuper)(&superStruct, sel, obj);
       
       NSLog(@"属性改变: %@", obj);
       // 任何自定义的通知方式
   }
   ```

   因为`objc_msgSendSuper`在系统头文件中的定义是`objc_msgSendSuper(void)`这样的，所以我们调用的时候需要手动转换一下函数指针的类型。<u>实际上我们在使用`super`关键字的时候最终都会被转换成`objc_msgSendSuper`的调用</u>

   调用完super方法之后下面就是实现我们自定义的通知观察者的方式了，可以用Block、Notification或是其他方式，这里就不再演示了。

