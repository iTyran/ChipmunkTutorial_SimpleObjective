-----  E.Y -----
#Simple Objective-Chipmunk Tutorial:


This tutorial aims to be a bare-bones introduction to using Objective-Chipmunk in an iPhone game. It is very heavily commented and should be easy to follow. Despite containing only 100 or so actual lines of code, it squeezes in an overview of a number of topics:

Creating a Chipmunk space to simulate objects in it
Creating a bouncing box with friction
Controlling gravity using the device's tilt
Using collision callbacks to make impact sounds based on how hard objects collide
Using collision callbacks to track if an object is sitting on the ground or in the air
Using the ChipmunkObject protocol to easily add complex objects to a space
Using a CADisplayLink for smooth animation
Integrating Chipmunk into a Cocoa Touch UI
Even if you are just going to use the vanilla C API, this tutorial serves as a nice introduction to Chipmunk on the iPhone.

You can download this tutorial and all the project files from the GitHub page. These are ready to build!

#What Do I Need to Know First?

This is a pretty basic tutorial, but it does not explain Objective-C memory management or the Cocoa Touch APIs. You'll have to know at least a little about them or be willing to look them up in Apple docs to follow along.

#What is Chipmunk?

Chipmunk is a 2D rigid body physics library distributed under the MIT license. It is intended to be fast, portable, numerically stable, and easy to use. For this reason it's been used in hundreds of games on just about every system you could name. This includes top quality titles such as Night Sky for the Wii and many #1 sellers on the iPhone App Store! I've put thousands of hours of work over many years to make Chipmunk what it is today. Check out Chipmunk's website for more information.

#What is Objective-Chipmunk:

Objective-Chipmunk is an Objective-C wrapper for the Chipmunk Physics Library. While Chipmunk's C API is pretty easy to use, the Objective-C API is even better. The primary advantages of a native Objective-C API include integrating with the Cocoa memory management model and the Chipmunk Object protocol. The Chipmunk Object protocol unifies the basic Chipmunk types as well as making it easy to create custom composite collections of the basic types. Additionally, the wrapper adds many convenience methods for doing common setup tasks as well as helper methods that integrate it with the rest of the Cocoa Touch API. The wrapper tries to do things the Objective-C way, adding useful method variations where it makes sense to do so.

You can find out more information on Objective-Chipmunk's webpage. While Objective-Chipmunk is not free like Chipmunk is, the enhanced API will almost certainly save you time and money. You'll also be helping to support further Chipmunk development!

#Why are you using Cocoa Touch classes for a game tutorial?

Cocoa Touch actually works great for prototyping an iPhone game quickly. It's reasonably fast because it's hardware accelerated, and doesn't seem to slow down too much until you have a few dozen objects on the screen. Interface Builder makes a decent bare bones level editor as well. We've written several contract mini-games this way and it saved a lot of headaches dealing with libraries and dependencies. Given the simplicity of the games, using anything fancier would have been a waste of time.

We have more example code available on the Chipmunk downloads page including Cocos2D examples.

##Let's get started!

A very valid question of many Chipmunk beginners is how to structure their game to take advantage of Chipmunk. A lot of people try to build their game around Chipmunk, but this isn't a very good idea. Chipmunk is not meant to be a game engine. It doesn't render any graphics, and you can't get a list of all the bullets or monsters currently in the scene.

To use Chipmunk, you should treat it as a component. When you add a game object to your game scene, add it's physics objects to the Chipmunk space you created for that scene. As you will see in this tutorial, the ChipmunkObject protocol is easy to implement, but makes it trivial to add and remove entire game objects from a Chipmunk space with a single method call. When you want to render a sprite, you can get read it's position and rotation from the Chipmunk object it is linked to. This is sort of an MVC approach applied to games, and it works quite well.

A lot of people do this the other way around. They loop over all the collision shapes in Chipmunk and update the sprite that it is associated with. While this works, it's not very flexible. It means that each collision shape has exactly one sprite associated with it and vice versa. Furthermore, the API to iterate the shapes in a Chipmunk space is not part of the public documented API so I wouldn't recommend using it.

The goal is to make a box that we can move around the screen by tilting the iPhone or tapping on the box. Going a bit further, we'll also use collision callbacks to change the screen color and play impact sounds. There are basically 2 classes: ViewContoller which we are using like a game controller, and FallingButton which we are using like a game object controller.

--- 夜狼 ---
#ViewController.m - A Bare Bones Game Controller

A game controller is responsible for for a couple of things. A game controller's main responsibility is to control the game's logic, to decide when the player wins or loses, that sort of thing. Our simple example doesn't really have any rules, so we'll skip straight to it's other responsibilities: managing the game loop and handling adding and removing game objects.

###Initialization and Setup:

We're building our game controller as a UIViewController, so let's take a look at the viewDidLoad method where the initialization happens. We'll go through it line by line.

First we need to initialize the graphics and the physics. Because we are writing this as a view controller, we can just use the view for graphics. All we need to do to set that up is call the viewDidLoad method in the super class:

      [super viewDidLoad];
      
Now we can add subviews to self.view and they will simply show up on the screen. That was easy.

###Initializing the physics is just as easy:

      space = [[ChipmunkSpace alloc] init];
      
The space is completely empty at this point. Anything other physics objects we add can simply fly off the screen if they wanted to. The first thing that a lot of 2D physics games do is to create a box around the screen. Objective-Chipmunk actually has a nice convenience method for that.

      [space addBounds:self.view.bounds
        thickness:10.0f
        elasticity:1.0f friction:1.0f
        layers:CP_ALL_LAYERS group:CP_NO_GROUP
        collisionType:borderType
      ];
thickness controls how thick the border is, layers and group control filtering of collisions. See the documentation on collision shapes for more information. Lastly, borderType is an object that we use as a key when defining collision callbacks. This lets you create a collision callback that is only called when a bullet hits a monster for example. The collision type can be any object. Class objects and global NSString objects work well as collision types as they are easy to get to. In this case, borderType was simply a global NSString that I defined at the top of the file.

Next we'll set up a collision callback to respond to collision events when the box touches the screen border.

      [space addCollisionHandler:self
        typeA:[FallingButton class] typeB:borderType
        begin:@selector(beginCollision:space:)
        preSolve:nil
        postSolve:@selector(postSolveCollision:space:)
        separate:@selector(separateCollision:space:)
      ];
There are 4 collision events that are sent to a collision handler. You only have to implement the ones that you are interested in. In this case, we want to know when shapes shapes start touching, when Chipmunk has resolved the collision, and when they stop touching. You can find out more from thecallback documentation. Also, the methods are only called when two shapes tagged with [FallingButton class] and borderType collide.

Lastly, we'll create the falling button game object and add it's view to the scene and add it's physics objects to the Chipmunk space.

      fallingButton = [[FallingButton alloc] init];
      [self.view addSubview:fallingButton.button];
      [space add:fallingButton];
      
Because FallingButton implements the ChipmunkObject protocol, we can add (or remove) all of it's rigid bodies, collision shapes and joints to the space in a single method call no matter how complex it's physics are.

That's it for the game controller initialization. Let's look at all the code together.

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
        
##The Game Loop: Updating the Physics and Graphics

So now we get to the meat of what the game controller does. Controlling the game's update loop. This tutorial uses a CADisplayLink to receive events from the iPhone OS when the screen wants to redraw itself. This is the easiest way to get smooth animation that I know of. Let's create a display link callback and an accelerometer callback quick:

        - (void)viewDidAppear:(BOOL)animated {
          displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(update)];
          displayLink.frameInterval = 1;
          [displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSDefaultRunLoopMode];
          
          UIAccelerometer *accel = [UIAccelerometer sharedAccelerometer];
          accel.updateInterval = 1.0f/30.0f;
          accel.delegate = self;
        }
        
These are Apple's classes, so I won't go into too much detail here.

Next we'll create the update method for the display link to call.
        
        - (void)update {
          cpFloat dt = displayLink.duration*displayLink.frameInterval;
          [space step:dt];
          
          [fallingButton updatePosition];
        }
        
This simply uses the timing information from the display link to figure out the amount of time passed since the last time the method was called (dt). We can then step (update) the physics simulation using that time. After the simulation has been updated, we update the falling button object so it's view lines up with the physics.

One thing to note here is that Chipmunk prefers it when you use the same dt each time you step the simulation. It's not required, but it will allow you to reduce the CPU usage significantly as well as making your game deterministic.

Next we'll define the accelerometer callback so that we can update gravity to respect the tilt of the iPhone.

        - (void)accelerometer:(UIAccelerometer *)accelerometer didAccelerate:(UIAcceleration *)accel {
          space.gravity = cpvmult(cpv(accel.x, -accel.y), 100.0f);
        }
        
Pretty easy. One thing to note is that all of the cpv*() functions are operators for Chipmunk vectors. cpv() creates a new vector from an x and y value, and cpvmult() multiplies a vector by a length. Seeing vector math done this way might take some getting used to. See the cpVect C docs for more information. If you are using Objective-C++ you can easily define overloaded operators for Chipmunk.

##Collision Callbacks: begin

Different collision callback types have slightly different method signatures. The begin callback method should look something like this:

        - (bool)beginCollision:(cpArbiter*)arbiter space:(ChipmunkSpace*)space

The first argument is a pointer to a C cpArbiter struct. The word arbiter means a judge that resolves a dispute. Because collision callbacks are trigger so often, I chose not to create a temporary Objective-C object to wrap the arbiter. Fortunately, there are only a few things you need to interact with cpArbiter structs for. For more information on arbiters, see the cpArbiter C docs.

The second argument is the ChipmunkSpace object that the callback was registered on. That way you can do things like register post-step (different than a post-solve callback) callbacks that add or remove objects from the space.

The first thing you probably want to do in a collision callback is probably to find out which two shapes collided. If you remember from earlier, we define our collision handler to be called when shapes with two specific collision types collided. So how do you figure out which object is which? Objective-Chipmunk provides a handy macro to define the ChipmunkShape variables as well as initialize them for you.

         CHIPMUNK_ARBITER_GET_SHAPES(arbiter, buttonShape, border);
         
That macro is equivalent to doing something like this:

          ChipmunkShape *buttonShape = GetShapeWithTypeA(arbiter);
          ChipmunkShape *border = GetShapeWithTypeB(arbiter);

Where type A and type B are the same as the types you gave to the space when you defined the collision handler.

A common thing that games need to know is when an object is touching the ground. Using Chipmunk's collision events, we can easily track this using a counter. Increment the counter in your begin callbacks, and decrement it in the separate callbacks. If the counter is 0, then you know the shape isn't touching anything. Let's do that here.

We have a problem though. The collision callback only gave us the colliding shape and not the game object it's for! Fortunately, Chipmunk provides data pointers on it's objects so that you can always get a reference to your game object from a rigid body, shape or joint. The data pointer is set by the FallingButton class when we create it. You'll see more about this later.

          FallingButton *fb = buttonShape.data;

So now we can use that to increment the counter on our falling button.

          fb.touchedShapes++;
          
Lastly, lets set the background color to gray when the falling button is touching anything so we can see it.

          self.view.backgroundColor = [UIColor grayColor];

Lastly, the begin callback must return a boolean. If it returns TRUE, then Chipmunk processes the collision normally. If it returns FALSE then Chipmunk ignores the collision as if it never happened. This is a powerful way to implement all sorts of conditional collision filtering such as one way platforms or breakable objects. We just want to process the collision normally, so we'll just return TRUE. Putting this all together, we have our finished begin callback method:

        - (bool)beginCollision:(cpArbiter*)arbiter space:(ChipmunkSpace*)space {
          CHIPMUNK_ARBITER_GET_SHAPES(arbiter, buttonShape, border);
        
          FallingButton *fb = buttonShape.data;
          fb.touchedShapes++;
          
          self.view.backgroundColor = [UIColor grayColor];
          
          return TRUE;
        }
        
#Collision Callbacks: separate

The separate callback has a slightly different method signature from the begin callback. Because the collision is already done by the time the separate callback is called, it doesn't return a boolean that controls whether to ignore the collision.
        
        - (void)separateCollision:(cpArbiter*)arbiter space:(ChipmunkSpace*)space
 
First we'll decrement the counter. This should all look familiar.
        
          CHIPMUNK_ARBITER_GET_SHAPES(arbiter, buttonShape, border);
          
          FallingButton *fb = buttonShape.data;
          fb.touchedShapes--;
          
Now let's set the background color to a random color so you can see each time the box collides with the border. We only want to do this when the box is no longer touching any of the 4 border edges, so we check if the counter is 0 first.

          if(fb.touchedShapes == 0){
            self.view.backgroundColor = [UIColor colorWithRed:frand() green:frand() blue:frand() alpha:1.0f];
          }
          
Putting this all together we have the finished separate callback:
        
        - (void)separateCollision:(cpArbiter*)arbiter space:(ChipmunkSpace*)space {
          CHIPMUNK_ARBITER_GET_SHAPES(arbiter, buttonShape, border);
          
          FallingButton *fb = buttonShape.data;
          fb.touchedShapes--;
          
          if(fb.touchedShapes == 0){
            self.view.backgroundColor = [UIColor colorWithRed:frand() green:frand() blue:frand() alpha:1.0f];
          }
        }
        
-----  wAe]ChildhoodAndy ------        
#碰撞回调：post-solve

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

#Falling Button:一个简单的游戏对象控制器

和游戏控制器类似，游戏对象控制器的主要功能就是管理游戏对象的逻辑更新并将图形和物理两者做绑定。在这篇教程中，falling button的逻辑比较简单。我们所要做的就是点击它的时候让它向随机的方向移动。

## 初始化设置：

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

为了乐趣所在，你应该尝试着看看不同的转动惯量值的影响，以及为什么使用Chipmunk提供的转动惯量估算函数很重要。你可以在`cpBody`C文件里找到更多的这些估算函数。

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
        
#按钮动作:

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

#结束语：

既然你瞧见了Objective-Chipmunk是多么容易并且强大，何不将它集成到你的iPhone游戏里面呢？Objective-C API提供的与流行iPhone库如Cocos2D协同良好的高层次API，将会为你省去不少时间（和金钱）。同样你也不必为内存管理担心因为它允许你像你的其他iPhoneApp一样来管理Chipmunk内存。

开心的玩转Chipmunk吧！

