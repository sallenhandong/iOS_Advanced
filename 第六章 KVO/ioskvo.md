![](https://user-gold-cdn.xitu.io/2018/8/18/1654b3082fe84cf4?w=1954&h=1050&f=jpeg&s=103789)
### IOS KVO原理解析与应用
#### 一、KVO概述
KVO，即：```Key-Value Observing```，是Objective-C对观察者模式的实现，每次当被观察对象的某个属性值发生改变时，注册的观察者便能获得通知，这种模式有利于两个类间的解耦合，尤其是对于业务逻辑与视图控制 这两个功能的解耦合。
#### 二、KVO有哪些应用？
- NSOperation
- NSOperationQueue
- RAC
#### 三、KVO的使用和实现？
#### 1、使用KVO
1.注册观察者，指定被观察对象的属性： 

[_people addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionOld | NSKeyValueObservingOptionNew context:nil];
2.在观察者中实现以下回调方法：
```
- (void)observeValueForKeyPath:(NSString *)keyPath  
ofObject:(id)object  
change:(NSDictionary *)change  
context:(void *)context  

{  
NSString *name = [object valueForKey:@"name"]; 
NSLog(@"new name is: %@", name);  
}  
```
只要People对象中的name属性发生变化，系统会自动调用该方法。

3.最后不要忘了在dealloc中移除观察者
```
[_people removeObserver:self forKeyPath:@"age"]; 
```
#### 2、KVO的实现
KVO 在apple文档的说明

```
Automatic key-value observing is implemented using a technique called 
isa-swizzling… When an observer is registered for an attribute of an object the 
isa pointer of the observed object is modified, pointing to an intermediate class 
rather than at the true class …
```
利用运行时，生成一个对象的子类，并生成子类对象，并替换原来对象的isa指针，重写了set方法。  
#### 让我们看看代码
- 这是我们创建的`Myprofile`类
```
@interface MyProfile : NSObject
@property (nonatomic,strong) NSString *avatar;
@property (nonatomic,strong) NSString *age;
@property (nonatomic,strong) NSString *name;
@property (nonatomic,strong) NSMutableArray *dataArr;
@property (nonatomic,strong) MyDetail *myDetail;
```
- 再看`viewcontroller`
```
self.myprofile = [[MyProfile alloc]init];
self.myprofile.name = @"sallen";
NSLog(@"before:%s",object_getClassName(self.myprofile));
[self.myprofile addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew context:nil];
self.myprofile.name = @"slim";
NSLog(@"after:%s",object_getClassName(self.myprofile));
```
#### 1.通过打印可以看出`class`明显发生了变化,监听之后的`class`替换了原有`class`的`isa`指针
![](https://user-gold-cdn.xitu.io/2018/8/18/1654affac0ca12a6?w=1314&h=478&f=jpeg&s=136805)
#### 2.再看看子类
```
self.myprofile = [[MyProfile alloc]init];
self.myprofile.name = @"sallen";
NSLog(@"before:%@",[self findSubClass:[self.myprofile class]]);
[self.myprofile addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew context:nil];
self.myprofile.name = @"slim";
NSLog(@"after:%@",[self findSubClass:[self.myprofile class]]);
```
通过打印可以看出明显多了个子类
![](https://user-gold-cdn.xitu.io/2018/8/18/1654b098a9eb0661?w=1292&h=486&f=jpeg&s=104990)
#### 3.对于容器的监听
```
[self.myprofile addObserver:self forKeyPath:@"dataArr" options:NSKeyValueObservingOptionNew context:nil];
[self.myprofile.dataArr addObject:@"slim"];
```
通过监听数组发现，是没有触发的通知的，因为重写了set方法。  
我们可以利用kvc实现对数组的监听  
```
[[self.myprofile mutableArrayValueForKeyPath:@"dataArr"] addObject:@"slim"];
```  
![](https://user-gold-cdn.xitu.io/2018/8/18/1654b14f25299ae6?w=1276&h=462&f=jpeg&s=99423)  
#### 4.多级路径属性
`Myprofile`类里又包含了`MyDetail`类  
`Mydetail`创建了`content`属性
如果我们需要监听`myDetail`属性的变化
我们在`Myprofile.m`通过方法：`+ (NSSet *)keyPathsForValuesAffectingValueForKey:(NSString *)key`，
一个Key观察多个属性值的改变。
```
+ (NSSet *)keyPathsForValuesAffectingValueForKey:(NSString *)key{

NSSet *keySet = [super keyPathsForValuesAffectingValueForKey:key];
if ([key isEqualToString:@"myDetail"]) {
NSSet *set = [NSSet setWithObject:@"_myDetail.content"];
keySet = [keySet setByAddingObjectsFromSet:set];
}

return keySet;
}
```  
打印结果:
![](https://user-gold-cdn.xitu.io/2018/8/18/1654b27bab329ea2?w=1158&h=338&f=jpeg&s=63551)
#### 四、KVO的缺陷
KVO很强大，但是也有缺点  
</br>
1.只能重写 `-observeValueForKeyPath:ofObject:change:contex `这个方法 来获得通知,不能使用自定义的`selector`， 想要传一个`block `更是不可能 ，而且还要处理父类的情况 父类同样观察一个同样的属性的情况 ，但是有时候并不知道父类 是不是对这个消息有兴趣。 
</br>
2.父类和子类同时存在KVO时，很容易出现对同一个`keyPath`进行两次`removeObserver`操作，从而导致程序`crash`。要避免这个问题，就需要区分出`KVO`是`self`注册的，还是`superClass`注册的，我们可以在 `-addObserver:forKeyPath:options:context:`和`-removeObserver:forKeyPath:context`这两个方法中传入不同的`context`进行区分。

#### 五、block方式的实现
##### 等待更新，
> * Follow: https://github.com/sallenhandong
> * Source: slimsallen.com/#/detail/ioskvo.md
