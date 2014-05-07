# 简明Objective-Chipmunk教程:

本教程的目标是简单的介绍下在iPhone游戏中使用Objective-Chipmunk。其中有很多注释，理解起来很容易。尽管代码只有100行左右，但我会从下面几个主题展开：
    
-  创建一个Chipmunk空间模拟对象
-  创建一个具有摩擦力的弹性盒子  
-  通过倾斜设备来控制重力
-  使用碰撞回调来实现基于物体碰撞力度的声音大小
-  根据碰撞回调来追踪物体处于地面还是空中    
-  使用ChipmunkObject protocol来轻松的将复杂对象加入到空间中
-  使用CADisplayLink实现平滑动画
-  整合Chipmunk进CocoaTouch UI

即使你打算只是用vanilla C API, 本教程也可当作一篇Chipmunk在iPhone上使用的不错的介绍。 

你可以在[GitHub网站](https://github.com/andykorth/SimpleObjectiveChipmunkTutorial)上下载本教程以及所有的工程文件。他们都是可以编译的！

## 我首先该了解什么呢？

这是一篇很棒的基础教程，但是并不打算介绍Objective-C内存管理或者Cocoa Touch APIs。你至少需要对他们有一些了解或者愿意查看苹果官方文档。

## Chipmunk是什么？

Chipmunk2D是一个基于MIT协议的2D刚体物理仿真库。设计宗旨:极快、可移植、稳定、易用。出于这个原因，它已经被用于数以百计的游戏，而且几乎横跨了所有系统。这些游戏包括了iPhone AppStore上一些顶级出色的TOP1游戏，如Night Sky等。这几年来，我投入了大量的时间来发展Chipmunk，才使得Chipmunk走到今天。更多信息请查阅[Chipmunk官网](http://chipmunk-physics.net/)。

## Objective-Chipmunk是什么？

Objective-Chipmunk是Objective-C对Chipmunk物理引擎库的封装。虽然Chipmunk的C API非常容易使用，但Objective-C API更棒。原生Objective-C API的主要优点包括集成了Cocoa内存管理模型和ChipmunkObject协议。Chipmunk对象协议统一了Chipmunk的基本类型。另外该封装增加了许多便利的方法来执行常见的安装任务，以及整合了Cocoa Touch API辅助方法。封装会尽量按照Objective-C的方式来实现，并在一些需要的地方添加一些有用的方法变量。

Chipmunk Object协议统一了基础Chipmunk类型，使得它很容易创建自定义的基础类型类型。另外，包装中添加了很多便捷的方法来执行场景的安装任务，。该包装尝试完成object-c的方式，添加更加有意义的方法变量等。

你可以在[Objective-Chipmunk网站](http://howlingmoonsoftware.com/objectiveChipmunk.php)上查阅更多信息。虽然Objective-Chipmunk并不免费，但是增强的API肯定会为你节约时间和成本。同时你也将会支持Chipmunk的发展！

## 为什么使用Cocoa Touch实现本教程

Cocoa Touch 实际上能够快速开发一款iPhone 游戏。因为硬件加速的原因会相当快，同时在屏幕上有许多对象也不会慢。Interface Builder也是一款简要的编辑器。我们用这个方式写了多款小游戏，并且让我们避免处理库的依耐性的麻烦。结合游戏的简洁性，使用任何华丽的空想都将是浪费时间。

在[Chipmunk下载页](http://chipmunk-physics.net/documentation.php)中，我们有更多可运行的例子包括Cocos2D实例。
    
# 让我们开始吧!

许多Chipmunk初学者有个疑问，就是如何利用Chipmunk的优势来构建他们的游戏。很多人尝试使用Chipmunk来建立他们的游戏，但这并不是好的主意。Chipmunk不是一个游戏引擎。它不显示图像，你也不能获得场景里的所有子弹和怪物。 

如果要使用Chipmunk，你应该把它作为一个组件。当你在游戏场景里添加游戏对象时，在你创建的场景里添加物理对象到Chipmunk空间里。本教程你可以看到，Chipmunk Object协议很容易实现，但是通过调用单个方法，可以对Chipmunk世界里的组件添加和删除。当你想显示一个精灵，你可以通过Chipmunk Object读取精灵的位置和旋转角度。这就是MVC应用于游戏的缩影，并且使用的非常好。

很多人反其道而行。他们通过遍历所有的碰撞并且更新相关的精灵。这样的工作模式，是不灵活的。它意味着每次碰撞只有一个精灵关联，反之亦然。此外，API迭代Chipmunk世界的形状不是公共明确的API的一部分，所以我不推荐使用它。

主要的目地就是在屏幕内完成一个可以通过倾斜iphone或者触摸移动的刚体。说白了我们通过碰撞回调方法来改变屏幕的颜色并且播放相应的音效。这里有两个基础类：我们喜欢用的一个游戏控制器:ViewContoller，以及游戏对象控制器:FallingButton。


## ViewController.m - 简单的游戏控制器

游戏控制器要负责很多事情， 一个游戏控制器最主要的是控制整个游戏的逻辑， 判定玩家什么时候赢什么时候输这类的事情。 我们简单的例子里面并没有任何规则， 所以我们直接跳到其它需要负责的方面：管理游戏循环和处理添加删除游戏对象

### 初始化设置：

我们用UIViewController来创建游戏的控制器(controller),来一起看一下在viewDidLoad函数中初始化的时候都做了什么，我将一行一行的进行解释

首先我们需要初始化图像和物理属性，我们要把这些卸载一个view controller里面， 我们可以仅用view来进行图像显示，所以我们只需要调用父类的viewDidLoad方法就可以完成初始化了。

```
[super viewDidLoad];
```
现在我们就可以把他们添加进来显示在界面上了，相当easy。

初始化物理世界：

```
space = [[ChipmunkSpace alloc] init];
```

新创建的space对象里面什么也没有。我们添加的任何物理对象都会飞出屏幕之外。一般使用物理引擎的2D的项目都会在开始的时候设置屏幕的边界，Objective-Chipmunk也有一个很好的简便方法来实现。

```
[space addBounds:self.view.bounds
       thickness:10.0f
       elasticity:1.0f friction:1.0f
       layers:CP_ALL_LAYERS group:CP_NO_GROUP
       collisionType:borderType
];
```

`thickness`控制着边界的厚度，`layers`和`group`控制碰撞过滤和碰撞。更多信息请查看[碰撞形状](http://files.slembcke.net/chipmunk/release/ChipmunkLatest-Docs/#cpShape)文档。
最后再说一下`borderType`是一个当定义了碰撞回调函数时的标记的对象，比如你想在子弹击中怪物的时候进行碰撞处理，碰撞的对象可以是任何对象，类的对象或者全局NSString都可以，我的borderType就是一个在头文件里面定义的全局NSString变量。

接下来我们设置一下物体碰到界面边界的时候的响应函数。

```
[space addCollisionHandler:self
        typeA:[FallingButton class] typeB:borderType
        begin:@selector(beginCollision:space:)
        preSolve:nil
        postSolve:@selector(postSolveCollision:space:)
        separate:@selector(separateCollision:space:)
];
```
总共有4种碰撞事件，你只需要找到一个感兴趣的实现就可以了，如果想知道什么时候两个物体碰撞、什么时候Chipmunk处理碰撞、什么时候结束碰撞 这些都可以通过查阅[响应函数文档](http://files.slembcke.net/chipmunk/release/ChipmunkLatest-Docs/#Callbacks)来了解

最后我们来创建一个掉落按钮的游戏对象，让它在view中显示，把它的物理属性添加到Chipmunk的space中
。

```
fallingButton = [[FallingButton alloc] init];
[self.view addSubview:fallingButton.button];
[space add:fallingButton];
```
      
因为FallingButton实现了ChipmunkObject协议，我们就可以直接添加或者删除它的所有刚体，形状(shapes)和关节(joints)的碰撞都是在同一个函数里面处理的，所以不需要考虑它的物理属性有多负载

这就是游戏controller的初始化，我们一起来看一下代码。

```
static NSString *borderType = @"borderType";
        
- (void)viewDidLoad {
	[super viewDidLoad];
          
	space = [[ChipmunkSpace alloc] init];
	[space addBounds:self.view.bounds
		thickness:10.0f
		elasticity:1.0f friction:1.0f
		layers:CP_ALL_LAYERS group:CP_NO_GROUP
		collisionType:borderType
	];
          
	[space addCollisionHandler:self
		typeA:[FallingButton class] typeB:borderType
		begin:@selector(beginCollision:space:)
		preSolve:nil
		postSolve:@selector(postSolveCollision:space:)
		separate:@selector(separateCollision:space:)
	];
          
	fallingButton = [[FallingButton alloc] init];
	[self.view addSubview:fallingButton.button];
	[space add:fallingButton];
}
```
        
### 游戏主循环:更新物理和图像

现在我们来看一下游戏的控制器（controller）都干了些什么，控制整个游戏的更新（update）循环。在这本节教程中用CADisplayLink来接收iPhone OS在屏幕重画是触发的事件，这是我知道的最简单的得到很流畅的动画的方式，让我们来快速的创建一个连接显示和加速（accelerometer）的回调函数：

```
- (void)viewDidAppear:(BOOL)animated {
	displayLink = [CADisplayLink displayLinkWithTarget:self
selector:@selector(update)];
	displayLink.frameInterval = 1;
	[displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSDefaultRunLoopMode];
          
	UIAccelerometer *accel = [UIAccelerometer sharedAccelerometer];
	accel.updateInterval = 1.0f/30.0f;
	accel.delegate = self;
}
```        
        
这是一个很普通的Apple类，所以我就不进行详细的解释了。

接下来我们要创建一个让displayLink调用的update方法

```
- (void)update {
	cpFloat dt = displayLink.duration*displayLink.frameInterval;
	[space step:dt];
          
	[fallingButton updatePosition];
}
```        
        
只是简单的定时得到显示连接来计算距离上一次函数调用的时间，我们可以使用同样的间隔时间更新物理引擎，在物理引擎更新完成之后，更新falling button的位置(显示的图片根据物理属性的位置进行调整)

有一点需要说明一下，Chipmunk不需要随时保持一致，这样可以降低你的CPU的使用次数，而且让你的游戏更为准确

然后我们创建一个accelerometer回调函数用来在iPhone倾斜的时候更新重力方向。

```
- (void)accelerometer:(UIAccelerometer *)accelerometer didAccelerate:(UIAcceleration *)accel {
	space.gravity = cpvmult(cpv(accel.x, -accel.y), 100.0f);
}
```
        
相当简单，解释一下cpv这个函数，它的功能是根据x和y坐标创建一个Chipmunk中的一个向量，cpvmult()方法是创建一个单位向量，想要知道更多关于向量的信息可以自己去查阅[cpVect C 文档](http://files.slembcke.net/chipmunk/release/ChipmunkLatest-Docs/#cpVect)，如果你是在使用Objectiv-C++就可以很容易的重载Chipmunk的运算符了。

### 碰撞回调:begin

不同的碰撞回调类型对应不同的方法签名，begin回调方法可以写成像下面这样：

```
- (bool)beginCollision:(cpArbiter*)arbiter space:(ChipmunkSpace*)space
```

第一个参数是一个指向cpArbiter结构体的指针，arbiter这个词在这里的意思是仲裁者，由于很容易会触发碰撞，所以我决定不创建一个Objective-C的类来封装arbiter，我们很少会用到cpArbiter的结构体，想要知道更多关于cpArbiter可以查阅[cpArbiter C 文档](http://files.slembcke.net/chipmunk/release/ChipmunkLatest-Docs/#cpArbiter)。

第二个参数是回调注册所在的ChipmunkSpace空间对象。这样你可以注册post-step回调（和post-solve回调不同）来向空间中添加或者删除对象。

可能你首先想在碰撞的回调函数里面做的事是找出来到底是哪两个物体碰撞了，如果你还记得以前说过的东西，当空间中两个特定碰撞类型的物体碰撞时，我们定义来碰撞处理函数。那么我们该如何知道哪个对象是哪个呢？Objective-Chipmunk提供来一个处理宏来定义ChipmunkSpace变量以及初始化他们。

```
CHIPMUNK_ARBITER_GET_SHAPES(arbiter, buttonShape, border);
```

这个宏相当于做了这样的事：   

```
ChipmunkShape *buttonShape = GetShapeWithTypeA(arbiter);
ChipmunkShape *border = GetShapeWithTypeB(arbiter);
```
typeA和typeB和你定义碰撞处理函数的定义的type一样。

游戏中常常要知道物体是否已经接触来地面。使用Chipmunk的碰撞事件，我们可以很容易的通过一个计数器来实现。在begin回调中，增加计数器，并且在seperate回调中，减少计数器。如果计数器为0，则意味着物体不再接触任何东西。我们从这里开始。

我们一直有个问题，就是碰撞的回调函数只会给我们传过来碰撞的shape而不是它对应的游戏里面的对象！幸好。Chipmunk中提供了一个数据指针指向使用它的对象以便于你在任何时候都能通过刚体、shape或者关节来得到游戏中的对象，这个对象指针是在我们创建FallingButton对象的时候赋值的，后面你可以看到详细的解释。

```
FallingButton *fb = buttonShape.data;
```

那么现在我们增加以下我们掉落按钮的计数：

```
fb.touchedShapes++;
```

我们背景设置为灰色，以便于看清下降的按钮碰撞到的任何东西：

```
self.view.backgroundColor = [UIColor grayColor];
```
begin回调函数必须要返回一个boolean的值，如果返回true,那么Chipmunk会以正常的方式处理碰撞，如果返回false，那么Chipmunk会当做这个碰撞没有发生，这个在处理碰撞过滤的时候非常有用比如单向平台或易碎物体。如果我们只需要正常碰撞，将返回值设置为TRUE。做完这些事之后，我们也就完成了我们的begin会调函数:

```
- (bool)beginCollision:(cpArbiter*)arbiter space:(ChipmunkSpace*)space {
	CHIPMUNK_ARBITER_GET_SHAPES(arbiter, buttonShape, border);
        
	FallingButton *fb = buttonShape.data;
	fb.touchedShapes++;
          
	self.view.backgroundColor = [UIColor grayColor];
          
	return TRUE;
}
```

### 碰撞回调:separate

separate的回调方法和begin回调方法有些不一样，因为在调用separate回调函数的时候碰撞已经结束了，所以它不需要返回一个boolean类型来标识是否要忽略碰撞。

```
- (void)separateCollision:(cpArbiter*)arbiter space:(ChipmunkSpace*)space
```
 
首先我们要减少计数，这个跟前面增加的差不多，很简单

```        
CHIPMUNK_ARBITER_GET_SHAPES(arbiter, buttonShape, border);
          
FallingButton *fb = buttonShape.data;
fb.touchedShapes--;
```
          
接下来，我们将背景设置为随机的一种颜色以便于知道物体与边界发生了碰撞，我们只在物体不再与四个边界接触的时候这么做，所以，我们先检测一下计数是不是0

```
if(fb.touchedShapes == 0){
	self.view.backgroundColor = [UIColor colorWithRed:frand() green:frand() blue:frand() alpha:1.0f];
}
```
          
做完这些我们的separate回调函数也就搞定了：

```
- (void)separateCollision:(cpArbiter*)arbiter space:(ChipmunkSpace*)space {
	CHIPMUNK_ARBITER_GET_SHAPES(arbiter, buttonShape, border);
          
	FallingButton *fb = buttonShape.data;
	fb.touchedShapes--;
          
	if(fb.touchedShapes == 0){
		self.view.backgroundColor = [UIColor colorWithRed:frand() green:frand() blue:frand() alpha:1.0f];
	}
}
```
              
### 碰撞回调：post-solve

另一个很常见的回调处理就是播放碰撞声音了。为了让声音听起来自然，我们要基于物体撞击的力度来设置音量的大小。这便是post-solve回调要解决的问题。Chipmunk已经完成了碰撞解决，使得你有机会取得施加的冲力。

post-solve回调方法签名如下：

```
- (void)postSolveCollision:(cpArbiter*)arbiter space:(ChipmunkSpace*)space
```
和begin、seperate回调只发生在碰撞的第一帧和最后一帧不同，pre-solve和post-solve回调在两个形状接触过程中被每帧调用。为了播放一个碰撞声音，我们只需要关心第一帧。Chipmunk提供了一个C函数，你可以使用仲裁者来测试这是否是第一帧两个形状发生了碰撞。如果不是第一帧，我们使用它来提前退出。

```
if(!cpArbiterIsFirstContact(arbiter)) return;
```
有了这样的方式，现在我们只需要计算出物体间碰撞的力度以及播放一个基于力度大小的声音。

```
cpFloat impulse = cpvlength(cpArbiterTotalImpulse(arbiter));
        
float volume = MIN(impulse/500.0f, 1.0f);
if(volume > 0.05f){
	[SimpleSound playSoundWithVolume:volume];
}
```        

`cpArbiterTotalImpulse()` 返回一个向量。我们只关心力的大小，所以我们使用`cpvlength()`来得到向量的大小。将我们计算的大小进行截断确保不能大于1.0，并且如果足够响亮就播放声音。简单！

整合上面的代码，我们得到了完整的post-solve回调：

```
- (void)postSolveCollision:(cpArbiter*)arbiter space:(ChipmunkSpace*)space {
	if(!cpArbiterIsFirstContact(arbiter)) return;

	cpFloat impulse = cpvlength(cpArbiterTotalImpulse(arbiter));

	float volume = MIN(impulse/500.0f, 1.0f);
	if(volume > 0.05f){
 		[SimpleSound playSoundWithVolume:volume];
	}
}
```
上面基本就是游戏控制器了。

## Falling Button:一个简单的游戏对象控制器

和游戏控制器类似，游戏对象控制器的主要功能就是管理游戏对象的逻辑更新并将图形和物理两者做绑定。在这篇教程中，falling button的逻辑比较简单。我们所要做的就是点击它的时候让它向随机的方向移动。

### 初始化设置：

让我们从初始化开始来了解下游戏对象的组成。它由一个初始化的UIButton实例启动。下面是非常标准的Cocoa程序：

```
button = [UIButton buttonWithType:UIButtonTypeCustom];
[button setTitle:@"Click Me!" forState:UIControlStateNormal];
[button setTitleColor:[UIColor blueColor] forState:UIControlStateNormal];
[button setBackgroundImage:[UIImage imageNamed:@"logo.png"]
	forState:UIControlStateNormal];
button.bounds = CGRectMake(0, 0, SIZE, SIZE);
        
[button addTarget:self action:@selector(buttonClicked)	forControlEvents:UIControlEventTouchDown];
```
相当简单。现在我们来为它设置物理特性。我们先定义按钮的质量和转动惯量。等下！你会问这里说的转动惯量是什么意思。这看你怎么想的了。如果你认为物体的质量描述了物体移动起来需要的力度，那么转动惯量就是让物体转动起来需要的力度。

```
cpFloat mass = 1.0f;
cpFloat moment = cpMomentForBox(mass, SIZE, SIZE);
```
        
你可以轻松的为质量赋值并不需要使用任何特定的单位。只要确保他们彼此关联对比的要有意义。如果一个汽车的质量为1，那么一只狗的质量应该为0.01或者类似小的值。因为场景中只有一个按钮，所以我们为它设置的质量大与小无所谓了。反应会如出一辙。另一方面，对转动惯量进行猜测或赋值是个糟糕的想法。假设SIZE是100，那么按照上述方式计算的转动惯量将是850！

为了乐趣所在，你应该尝试着看看不同的转动惯量值的影响，以及为什么使用Chipmunk提供的转动惯量估算函数很重要。你可以在[cpBody  C 文档](http://files.slembcke.net/chipmunk/release/ChipmunkLatest-Docs/#cpBody)里找到更多的这些估算函数。

下面，我们来使用计算好的质量和转动惯量来初始化一个刚体：

```
body = [[ChipmunkBody alloc] initWithMass:mass andMoment:moment];
```
刚体拥有物理对象的物理属性。比如质量、位置、速度和角度。一旦你创建了个刚体，你便可以将任意数量的碰撞形状和关节与之关联。刚体有一些读写的属性，比如pos，vel，mass等等。当第一次创建的时候，除了质量和转动惯量外，大部分这些属性都为0。让我们将形状的位置设置在屏幕中间的某处。
    
```
body.pos = cpv(200.0f, 200.0f);
```
现在我们到了有趣的环节。为刚体创建一个碰撞形状使得它可以与屏幕边界形状进行碰撞。Chipmunk目前支持3种碰撞形状：圆形、线段和多边形。它具有便利的构造函数用来创建盒子状多边形。我们将使用它。因为我们不需要将物体存储为一个实例变量，所以我们使用autorelease构造函数。

```
ChipmunkShape *shape = [ChipmunkPolyShape boxWithBody:body width:SIZE height:SIZE];
```
这将创建一个长宽为SIZE的方形形状并关联到我们的刚体上。上面那个便捷的构造函数会将box形状居中放置到时刻与刚体位置保持一致的刚体重心位置上。重心也是刚体旋转所围绕的点。
        
如`ChipmunkBody`对象一样，`ChipmunkShape`对象也有一些你可能需要设置的属性：

```
shape.elasticity = 0.3f;
shape.friction = 0.3f;
```
`elasticity`控制着刚体有多大的弹性。当两个形状碰撞时，他们的弹性值会被相乘得到碰撞的弹性值。0意味着没有弹性，1意味着完全反弹。你可以设置大于1.0的值，意味着物体在每次碰撞之后会加速，可能会超出我们的控制。`friction`工作机制相同，两个形状的摩擦力相乘决定了最终施加的摩擦力。你可以从斜坡面的角度来考虑摩擦力值。如果摩擦力值是1，那么物体将会停止在45度的斜坡上。如果摩擦力值是0.5，那么物体将会停止在Atan(0.5) = 26.6度的斜坡上。
    
下面我们来设置在碰撞回调中要使用到的属性：

```
shape.collisionType = [FallingButton class];
shape.data = self;
```
如果你还记得上面，`collisionType`在创建碰撞回调被用来当作关键值，`data`属性可以引用回我们的falling button对象。

最后，我们需要实现`ChipmunkObject`协议以便我们不必再手动添加碰撞形状和刚体到空间中去。要做到这一点，我们需要做的就是实现一个单一的方法，该方法返回一个NSSet，它包含了这个游戏对象使用的所有基本的Chipmunk对象。最简单的方式就是使用`synthesized`属性。然后，你需要做的就是创建一个名为`chipmunkObjects`的实例变量并且使用`ChipmunkObjectFlatten()`函数来初始化它。你可以传递任何实现ChipmunkObject协议的值给`ChipmunkObjectFlatten()`函数。别忘记nil终止符和retain它返回的set结合。

```
chipmunkObjects = [ChipmunkObjectFlatten(body, shape, nil) retain];
```

将上面的整合一下，我们的初始化代码如下：

```
- (id)init {
	if(self = [super init]){
		button = [UIButton buttonWithType:UIButtonTypeCustom];
		[button setTitle:@"Click Me!" forState:UIControlStateNormal];
		[button setTitleColor:[UIColor blueColor] forState:UIControlStateNormal];
		[button setBackgroundImage:[UIImage imageNamed:@"logo.png"] forState:UIControlStateNormal];
		button.bounds = CGRectMake(0, 0, SIZE, SIZE);
            
		[button addTarget:self action:@selector(buttonClicked) forControlEvents:UIControlEventTouchDown];
            
		cpFloat mass = 1.0f;
		cpFloat moment = cpMomentForBox(mass, SIZE, SIZE);
            
		body = [[ChipmunkBody alloc] initWithMass:mass andMoment:moment];
		body.pos = cpv(200.0f, 200.0f);
            
		ChipmunkShape *shape = [ChipmunkPolyShape boxWithBody:body width:SIZE height:SIZE];
		shape.elasticity = 0.3f;
		shape.friction = 0.3f;
		shape.collisionType = [FallingButton class];
		shape.data = self;
            
		chipmunkObjects = [ChipmunkObjectFlatten(body, shape, nil) retain];
	}
          
	return self;
}
```      
        
### 按钮动作:

剩下唯一的事情就是处理点击按钮时所调用的方法了。

`frand_unit`函数
```
static cpFloat frand_unit(){return 2.0f*((cpFloat)rand()/(cpFloat)RAND_MAX) - 1.0f;}
```

```
- (void)buttonClicked {
	cpVect v = cpvmult(cpv(frand_unit(), frand_unit()), 300.0f);
	body.vel = cpvadd(body.vel, v);
          
	body.angVel += 5.0f*frand_unit();
}
```

没什么太复杂的内容。我们只是为刚体的速度和角速度添加了一个随机的改变。

## 结束语：

既然你瞧见了Objective-Chipmunk是多么容易并且强大，何不将它集成到你的iPhone游戏里面呢？Objective-C API提供的与流行iPhone库如Cocos2D协同良好的高层次API，将会为你省去不少时间（和金钱）。同样你也不必为内存管理担心因为它允许你像你的其他iPhoneApp一样来管理Chipmunk内存。

开心的玩转Chipmunk吧！

