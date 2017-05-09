# 入门指南

## 前言

    之前接触了CoreData，按照教程糊糊涂涂的实现了简单的操作，觉得这个框架挺麻烦的，后来一直没有涉及到就没有再用了。最近公司里要做一个技术分享，让我分享MagicalRecord框架 O__O。这个框架封装了Core Data复杂的操作，我觉得在使用框架前应该先把相关知识弄明白，所以就学习了Core Data相关的内容并做如下总结。

`ps：本篇文章主要侧重入门使用与简单的概念介绍，深入了解关于Core Data多线程的内容，一些细节性的知识点比如建立关系、分页查询、undo/redo、加锁等功能没有涉及到，请自行查阅相关资料。`
`内容如有叙述或理解错误，还请帮忙指正，谢谢。`

## 目录

> * 什么是Core Data
>     
>     
> * Core Data栈
>     
>     
> * 单线程实现增删改查
>     
>     
> * 使用简单的多线程Core Data
>     
>     
> * MagicalRecord框架使用的多线程方案
>     
>     
> * 轻量级数据迁移
>     
>     
> * 参考资料

* * *

## 什么是Core Data

### 简介

    Core Data是OS X 10.4 Tiger之后引入的一个持久化技术，通过与数据库进行交互，将模型的状态持久化到磁盘。
    使用了对象-关系映射（[ORM](http://baike.baidu.com/link?url=i3Z3H3r3piHjLwl-4z8HMWJa4eqLESH956tVpLtGPeocG2elUhgg4tHwGKELCu5f4-3y0tEH3uGwwIpF-5WTKa)）技术，很好的将数据中的表和字段转化为对象和属性，同时将表之间的关系转化成了对象之间的包含关系。
    我们可以轻易的操作这些对象实现增删改查功能，而这些操作Core Data都已经帮我们封装好了，它可以自动的将改变同步到数据库中。
    更易见的特点是它可以进行可视化操作，我们可以通过可视化的界面很容易的的管理实体、属性及实体关系等。

### 优点

```
> 1\. 封装性好，不需要使用任何的SQL语句就能够实现数据持久化的功能。
> 2\. 使用可视化的界面可以方便的进行实体管理和关系维护。
> 3\. 通过使用相应的封装类可以实现简单的性能管理，比如使用NSFetchRequest类进行查询。
> 4\. 支持多线程操作。
> 5\. 提供撤销重做（undo/redo）功能。
> 6\. 使用了一些简单的优化机制。比如有关联关系的两个实体类，被包含对象是以懒加载的方式进行实例化。
> 7\. 提供数据库迁移的功能。如果实体或者关系在app版本更新时有所改动，那么以前的数据将会被清空，这时就需要使用数据迁移去解决此问题。

```

### 缺点

```
> 1\. 概念多且比较难理解。
> 2\. 多线程Core Data需要做一些优化和处理。
> 3\. 难以实现一些sql能够简单实现的操作。比如多表连接查询等。

```

* * *

## Core Data栈

#### 我们先来通过建立一个项目来接触一下Core Data中的相关概念吧。

**1.** 新建项目，File -> New -> Project，这里我选择创建一个iOS单视图项目（Single View Application），输入项目名。`要记得勾选Use Core Data`。

![](https://segmentfault.com/img/bVrRLW)

**2.** 打开AppDelegate.h，我们可以看到使用Core Data相比不使用来说，在AppDelegate中会多出来三个成员属性和两个方法。

![](https://segmentfault.com/img/bVrROJ)

同时目录中也多出来一个与项目名称相同的.xcdatamodeld文件

![](https://segmentfault.com/img/bVrSaA)

`你可能不清楚这些类与文件代表什么，有什么作用，接下来就为你一一的解读。`

**3.** 下面就到了痛苦的概念理解了，但是明白这几个概念对于了解Core Data栈的建立和对于Core Data多线程的理解会有很大帮助。

`ps：以下内容请结合Core Data栈图`

#### Core Data中几个关键概念它们之间相互关联的结构被称做Core Data栈。

一般使用的Core Data栈，如图：
![](https://segmentfault.com/img/bVrRJW)

> * Entity 实体
>     
>     
> 
> > 我们可以在.xcdatamodeld文件对应的可视化界面中创建一些实体，并为其添加相应的属性和关系。这些实体和属性很类似于关系数据库中表和字段的概念，这是因为它是对象-关系映射中对象的部分，实体和属性与数据库中的表和字段相互映射。
> 
> * NSEntityDescription 实体对象描述
>     
>     
> 
> > NSEntityDescription描述了一个实体的基本属性，包括：实体名、类名、属性、关系等。我们可以通过实体描述来实例化`托管对象`。
> 
> * NSManagedObjectModel 托管对象模型 （MOM）
>     
>     
> 
> > 是实体对象描述（NSEntityDescription）的集合。通过.mom或.momd文件来进行实例化，其中.mom或.momd文件是由.xcdatamodeld文件编译后得到的`（想想看.xcdatamodeld文件是做什么事情的：通过可视化界面创建实体、添加属性、设置关系等）`。
> 
> * `NSManagedObject 托管对象 （MO）`
>     
>     
> 
> > `在我看来实体（Entity）像是一个模型，实体对象描述（NSEntityDescription）记录了这个模型的内容，对这个模型进行描述，而MO才是我们真正要去操作的东西。每一个MO都对应着一个实体且都有唯一的ID，因此它能够对应到数据库中的某一条记录。这样我们就可以对MO进行各种各样的操作，同时数据库中与之对应的记录会同步这些操作，这样就实现了ORM中的对象映射关系。`
> 
> * `NSManagedObjectContext 托管对象上下文 （MOC）`
>     
>     
> 
> > `那么问题来了，我们对MO的操作是如何被数据库知道，并进行同步的呢？这就需要用到MOC了。有没有觉得很奇怪，为什么我们操作的对象是“托管”的呢？这是因为我们使用了MOC去监听MO。作为监听者，当MO发生改变时，MOC会知道它所监听的MO对象发生了变化，然后就可以将这些改变提交给PSC，PSC将同数据库打交道来实现数据的同步。`
> 
> * NSPersistentStoreCoordinator 持久化存储协调器 （PSC）
>     
>     
> 
> > PSC使用NSPersistentStore对象与数据库进行交互，NSPersistentStore对象会将MOC提交的改变同步到数据库中。PSC是通过MOM进行实例化的，一方面是因为它需要与数据库和上下文交互，所以需要知道所有的实体描述才能正确的传达信息；二是因为在PSC初始化时，它会检测指定路径（一般为沙盒的Document文件夹下）中是否有相应的数据库，如果没有则进行创建，所以它需要知道所有的实体描述来创建数据库。同时如果指定路径中存在数据库，那么它会将最新的MOM与当前数据库进行比对，看是否存在最新的MOM中的实体与属性等与沙盒中数据库的表和字段等不一致的情况，（如果不一致，会创建新的数据库取代老数据库，老数据库中的数据将会丢失，这也是为什么我们需要做数据迁移的操作了）。

#### `总结来说其实就是：当我们想要向数据库插入一条数据时就需要将待保存的数据信息映射至MO，或者当我们需要修改或删除一条数据时需要获取到MO进行操作（该MO是从数据库中取出的数据经过映射后得到的MO），而我们create，query去获取到的这些MO均是被相应MOC进行监听，MOC会监听MO上的所有数据改变，然后在我们操作完MO之后，使用MOC将MO进行的改变提交给PSC，然后PSC操作数据库进行数据同步操作。`

![](https://segmentfault.com/img/bVyCHZ)

**4.** 还记得我们创建项目时AppDelegate中的那三个成员属性吗？那么了解了Core Data中的相关的概念后，就让我们趁热打铁来看一看系统为我们创建的Core Data栈的代码吧。

按照初始化顺序：MOM -> PSC -> MOC    NSManagedObjectModel的getter方法：

![](https://segmentfault.com/img/bVrSg3)

    NSPersistentStoreCoordinator的getter方法：

![](https://segmentfault.com/img/bVrShn)

    NSManagedObjectContext的getter方法：

![](https://segmentfault.com/img/bVrShs)

    contextSave方法：

![](https://segmentfault.com/img/bVrShX)

有没有觉得很简单，根据对上面一些概念的理解瞬间看懂了。^_^

`其实你也可以不用在创建项目的时候勾选Use Core Data选项，自己手动实现整个过程。通过file -> new -> file-> Core Data -> Data Model 创建一个自己的 .xcdatamodeld文件，接着在AppDelegate.h中导入头文件#import <CoreData/CoreData.h>，然后自己用代码尝试创建一个Core Data栈。`

* * *

## 单线程实现增删改查

明白了相关概念之后，我们就可以使用系统为我们创建的Core Data栈来实现数据持久化了。

### 准备工作

**1.** 使用系统自动创建的ViewController，为了能够使用AppDelegate中的MOC对象，在ViewController.m中 ViewController的扩展里添加AppDelegate的引用：

![](https://segmentfault.com/img/bVrTW0)

**2.** 在ViewController中添加控件并拖线，并初始化成员属性：myAppDelegate。如图：

![](https://segmentfault.com/img/bVrUEG)

**3.** 实体就使用前面创建的User实体，里面有两个属性分别为：uId和uName

### 增

让我们来想一下过程：

1. 创建MO对象，并使用MOC监听该对象

2. 操作MO，使MO发生改变

3. 保存，提交改变

![](https://segmentfault.com/img/bVrUD1)

其中创建MO对象有多种方法，你也可以使用NSManagedObject的类方法传参NSEntityDescription对象进行实例化。

### 查All

为了方便观察结果，我们先来做一个不带任何条件的简单查询吧，获取到所有User表记录对应的MO。

![](https://segmentfault.com/img/bVrUGS)

增加一条记录，然后查All，结果如图：
![](https://segmentfault.com/img/bVrUGO)

### 查

那么如果我们想要按照一定的条件进行查询呢？这就需要用到NSPredicate类来设置我们的查询条件。

![](https://segmentfault.com/img/bVrUWy)

其实主要就是加了两句代码，对于NSPredicate的使用百度上有很多介绍。这里是苹果官网的地址：
[https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/Predicates/Articles/pCreating.html](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/Predicates/Articles/pCreating.html)

### 删

学会了增加，同样的让我们先来想一下删除的过程。既然要删除，那么数据库中应该是有你想要删除的记录的，所以就需要先用到查询。

1. 查询出你想要删除的记录对应的MO

2. 使用MOC删除此MO

3. 保存，提交改变

![](https://segmentfault.com/img/bVrYCc)

### 改

其实也不用再多说了，就是三步：查询，设值，保存。

![](https://segmentfault.com/img/bVrYCF)

`把以上的步骤做完之后，其实CoreData简单的增删改查也已经实现了，明白了概念之后，CoreData其实也没有那么难，不是么？`

* * *

## 使用简单的多线程Core Data

### 为何使用多线程

    到了这里您一定会问，增删改查功能已经实现了，也用的好好的，为什么要使用多线程的Core Data呢？其实可以想一想，Core Data毕竟是对数据进行持久化的技术，如果数据量大的话，使用主线程操作必定会产生线程拥塞，而UI的更新就是在主线程中进行的，这会导致你的app界面“卡住”；另外当你需要同时执行多个操作时也需要使用多线程的Core Data。

### 如何实现多线程：一个线程使用一个NSManagedObjectContext对象

    至于怎么去实现，你首先可能会想到的是：实例化一个MOC对象，在几个子线程中分别去使用它执行增删改。但是这样是不行的，由于MOC不是线程安全的，如果多线程共用MOC的话会出现数据混乱，甚至更严重的会导致程序崩溃。此外MO也不是线程安全的，但是PSC是线程安全的，因为Context将改变提交给PSC时，系统会为其上锁，确保一次只执行一个MOC提交的改变。
    苹果官方给出的解决方法是：`一个线程使用一个MOC对象`，并且在该线程中只能操作它所监听的MO。

### 实例化NSManagedObjectContext

    让我们先来看一下MOC的带参init方法：

```
NSManagedObjectContext: - (instancetype)initWithConcurrencyType:(NSManagedObjectContextConcurrencyType)ct NS_DESIGNATED_INITIALIZER  NS_AVAILABLE(10_7,  5_0);

```

    其中的参数NSManagedContextConcurrencyType是一个枚举类型，我们来看一下源代码中这个枚举类型的各个枚举值。

```
typedef NS_ENUM(NSUInteger, NSManagedObjectContextConcurrencyType) {
    NSConfinementConcurrencyType  NS_ENUM_DEPRECATED(10_4,10_11,3_0,9_0, "Use another NSManagedObjectContextConcurrencyType") = 0x00,
    NSPrivateQueueConcurrencyType        = 0x01,
    NSMainQueueConcurrencyType            = 0x02
} NS_ENUM_AVAILABLE(10_7,  5_0);

```

    NSConfinementConcurrencyType在iOS5.0之后被废弃，新增了后面两个类型。

`NSConfinementConcurrencyType：每一个线程使用一个该类型的上下文，需手动开线程实现并发。`
`NSPrivateQueueConcurrencyType和NSMainQueueConcurrencyType：初始化时已为其绑定一个后台线程和主线程，不需要手动开线程。`

`后两种类型对应的MOC对象，执行操作时需使用performWithBlock:（对应着后台线程，block在后台线程中异步执行）和performWithBlockAndWait:（对应着主线程，block在主线程中同步执行）方法，`

```
NSManagedObjectContext: - (void)performBlock:(void (^)())block NS_AVAILABLE(10_7,  5_0);
NSManagedObjectContext: - (void)performBlockAndWait:(void (^)())block NS_AVAILABLE(10_7,  5_0);
```

`其中的block参数在对应绑定的线程中执行，所以context上的所有操作都需要写在block中，以避免跨线程导致的错误。`

### 简单的多线程方案

#### 1\. 方案一

使用两个MOC，一个负责在后台处理各种耗时的操作，一个负责与UI进行协作。

![](https://segmentfault.com/img/bVtkuS)

    我们知道MOC和MO不是线程安全的，为了解决这个问题我们在一个线程中仅使用一个MOC，因此不能跨线程访问同一个MOC和MO。但是这会存在问题。
    比如：我在后台线程中执行update操作，主线程中的上下文是不知道的，所以当我在主线程中去进行查询时，查询出来的结果并不是已更新后的结果。
`为了解决这个问题，我们需要监听通知：NSManagedObjectContextDidSaveNotification，在耗时操作处理完之后去告诉主上下文发生了哪些改变。我们可以通过主线程执行合并操作来实现。`

```
NSManagedObjectContext：- (void)mergeChangesFromContextDidSaveNotification:(NSNotification *)notification NS_AVAILABLE(10_5, 3_0);

```

##### 代码

初始化：
![](https://segmentfault.com/img/bVrZ1G)
增：
![](https://segmentfault.com/img/bVrZ1o)
删：
![](https://segmentfault.com/img/bVrZ2W)
改：
![](https://segmentfault.com/img/bVrZ3O)
查：
![](https://segmentfault.com/img/bVrZ3F)
通知合并操作：
![](https://segmentfault.com/img/bVrZ4c)

#### 2\. 方案二

iOS5.0之后新增了MOC之间的父子关系，使得管理context变得更为方便起来。

```
NSManagedObjectContext: @property (nullable, strong) NSManagedObjectContext *parentContext NS_AVAILABLE(10_7,5_0);
```

我们可以通过setParentContext方法来设置当前的MOC的父MOC。这样的好处是，只需要设置根MOC的PSC，而其下的所有子MOC均不用设置。`当子MOC提交改变的时候，会把改变提交给上一级MOC，直到传递给根MOC，由根MOC统一提交给PSC。`这样当一个线程中的MOC提交改变时就可以不用通知其他线程的MOC进行合并操作了，因为他们之间建立了父子关系。

![](https://segmentfault.com/img/bVr0oJ)

我们将使用三层的MOC去实现多线程Core Data，`privateContext -> mainContext -> rootContext`。
其中privateContext为我们操作的上下文对象，mainContext与UI协作，rootContext在后台执行费时操作。

##### 存在的问题

MO都有唯一的MOID与之对应，为了避免实例化MO时消耗大量资源来确保ID的唯一性，所以MO在实例化时会被给予一个临时的ID，这个ID在MOC范围内唯一。当MOC进行提交时，需要做一步临时ID转化为全局ID的操作，此时我们需要监听另外的一个通知：NSManagedObjectContextWillSaveNotification，然后对ID进行转化。

```
NSManagedObjectContext：- (BOOL)obtainPermanentIDsForObjects:(NSArray<NSManagedObject *> *)objects error:(NSError **)error NS_AVAILABLE(10_5, 3_0);

```

##### 代码

初始化：
![](https://segmentfault.com/img/bVr0nE)

私有方法：
![](https://segmentfault.com/img/bVr0nR)

![](https://segmentfault.com/img/bVr0nZ)

![](https://segmentfault.com/img/bVr0nQ)

增：

![](https://segmentfault.com/img/bVr0oh)

删：

![](https://segmentfault.com/img/bVr0og)

改：

![](https://segmentfault.com/img/bVr0oe)

查：

![](https://segmentfault.com/img/bVr0ob)

* * *

## MagicalRecord框架使用的多线程方案

MagicalRecord是对CoreData进行了一次封装，封装了多线程Core Data中繁琐的操作，使代码看起来更为清晰简洁。

##### 方案图

![](https://segmentfault.com/img/bVsXX8)
![](https://segmentfault.com/img/bVsXYd)

##### 对应代码

![](https://segmentfault.com/img/bVsXYy)

* * *

## 轻量级数据迁移

在做app版本的迭代过程中，难免会遇到要修改.xcdatamodeld文件，比如新增或删除一个实体、增加或删除一个原有实体的属性等。如果你没有设置数据迁移的话，app更新后原有的数据将会被清空，这当然是不行的，所以此时需要进行数据的迁移操作。core data可以设置轻量级的数据迁移，系统自动会帮你分析差异，进行映射，这种方式只适用于简单的增删实体或是增删属性等操作。除此之外还有一种相当复杂的自定义数据迁移，一般来说不会用到，本文不打算进行说明。

**1.**在Core Data栈中设置自动迁移功能

在PSC的实例化方法中添加自动迁移的相关设置。
![](https://segmentfault.com/img/bVyFj6)

如果你使用了MagicalRecord，只需要将之前初始化CoreDataStack的方法setupCoreDataStack 修改成使用 setupAutoMigratingCoreDataStack进行初始化。

![](https://segmentfault.com/img/bVyFqu)

**2.**添加新的CoreData版本，并切换到新版本

选中.scdatamodeld文件后，依次点击菜单项中的Editor->Add Model Version...
![](https://segmentfault.com/img/bVyFit)

此时.xcdatatmodeld文件就可以展开看到其包含的多个版本
![](https://segmentfault.com/img/bVyFiV)

根据下图所示的步骤即可将当前版本切换至你想要的版本
![](https://segmentfault.com/img/bVyFjk)

此时就可以在新版本上进行修改了
![](https://segmentfault.com/img/bVyFja)

ps：

1. 开启轻量级自动迁移之后，app每次启动都会判断实体与数据库表结构是否存在差异，如果存在差异，则会更新数据库文件，更新为新的表结构。

2. 如果在原有实体上增加了新属性，则迁移后的数据中该字段为空；如果在原有实体上减少了属性，那么迁移后的数据中该字段会被删掉，对应的数据也会被删除，即使你再切回到原始的版本，数据也不会恢复。

