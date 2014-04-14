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
        
#Collision Callbacks: post-solve

Another pretty common thing to do with callbacks is to play impact sounds. To make it sound good, we want to set the volume of the sound based on how hard the objects hit. This is exactly the sort of thing that the post-solve callback is for. Chipmunk has finished resolving a collision, and it gives you chance to read back the impulse it applied to do it.

The post-solve callback method signature looks like the separate one:

          - (void)postSolveCollision:(cpArbiter*)arbiter space:(ChipmunkSpace*)space

Unlike the begin and separate callbacks which just happen on the first and last frames of a collision, the pre-solve and post-solve callbacks are called every frame that the two shapes are touching. For playing a collision sound, we really only care about the first frame. Chipmunk provides a C function that you can use on the arbiter to test if this is the first frame that to shapes have been colliding. We'll use that to exit early from the function if it's not the first frame.

        if(!cpArbiterIsFirstContact(arbiter)) return;
        
With that out of the way, now we just need to find out how hard the objects hit and play a sound based off of that.

        cpFloat impulse = cpvlength(cpArbiterTotalImpulse(arbiter));
        
        float volume = MIN(impulse/500.0f, 1.0f);
        if(volume > 0.05f){
        [SimpleSound playSoundWithVolume:volume];
        }

cpArbiterTotalImpulse() returns a vector. We only really care about the size of the force, so we use cpvlength() to get the length of the vector. From that we calculate a volume, clamp it so that it's no bigger than 1.0 and play the sound if it's loud enough! Easy!

Putting that all together, we have our completed post-solve callback:

        - (void)postSolveCollision:(cpArbiter*)arbiter space:(ChipmunkSpace*)space {
          if(!cpArbiterIsFirstContact(arbiter)) return;
          
          cpFloat impulse = cpvlength(cpArbiterTotalImpulse(arbiter));
          
          float volume = MIN(impulse/500.0f, 1.0f);
          if(volume > 0.05f){
            [SimpleSound playSoundWithVolume:volume];
          }
        }
That's pretty much it for the game controller.

#Falling Button: A Simple Game Object Controller

Like a game controller, the game object controller's main responsibility is to manage the logic updates for a game object and bind the graphics and physics together. In this tutorial, the logic of the falling button is simple. All we want it to do is to move in a random direction when you click on it.

###Initialization and Setup:

Lets start with the initialization again to get a feel for what the game object is made of. It starts off with initializing a UIButton instance. Pretty standard Cocoa programming here:

        button = [UIButton buttonWithType:UIButtonTypeCustom];
        [button setTitle:@"Click Me!" forState:UIControlStateNormal];
        [button setTitleColor:[UIColor blueColor] forState:UIControlStateNormal];
        [button setBackgroundImage:[UIImage imageNamed:@"logo.png"] forState:UIControlStateNormal];
        button.bounds = CGRectMake(0, 0, SIZE, SIZE);
        
        [button addTarget:self action:@selector(buttonClicked) forControlEvents:UIControlEventTouchDown];
    
Pretty easy. Now we set up the physics for it. Lets start by defining the mass and moment of inertia of the button. Wait! What is this moment of inertia that I'm talking about you ask? It sounds completely made up you say! If you think of the mass of an object as being how hard it is to move an object around, the moment of inertia is how hard it is to spin an object.

        cpFloat mass = 1.0f;
        cpFloat moment = cpMomentForBox(mass, SIZE, SIZE);
        
You can easily just make up numbers for the mass as you don't need to use any particular units. Just make sure that they make sense in relation to each other. If a car has a mass of 1, then a dog should have a mass of 0.01 or something like that. With the button being the only thing on the screen, it doesn't really matter what mass we give it. It would react exactly the same. On the other hand, guessing or making up numbers for the moment of inertia is a really bad idea. Assuming that SIZE is 100, then the moment of inertia calculated above will be 850!

Just for the fun of it, you should try seeing what other values for the moment of inertia does, and why it's important to use the moment estimation functions that Chipmunk provides. You can find more of these estimation functions in the cpBody C docs.

Next, we'll use the calculated mass and moment of inertia to make a rigid body:

        body = [[ChipmunkBody alloc] initWithMass:mass andMoment:moment];
    
Rigid bodies hold the physical properties of a physics object. Things like it's mass, position, velocity and rotation. Once you've created a rigid body, you can attach any number of collision shapes and joints to it. Bodies have a number of properties that you can read and write such as pos, vel, mass and so on. When first created, most of these properties other than the mass and moment of inertia will be zero. Let's set the position of the shape to be somewhere in the middle of the screen.

        body.pos = cpv(200.0f, 200.0f);
        
Now we can get to the interesting part and create a collision shape for it so it will collide with the screen boundary shapes. Chipmunk supports 3 collision shapes at the moment. Circrles, line segments, and polygons. It also has a convenience constructor for creating box shaped polygons. We'll use that. Because we aren't storing the object in an instance variable, we'll use the autorelease constructor. (More on this later.)

        ChipmunkShape *shape = [ChipmunkPolyShape boxWithBody:body width:SIZE height:SIZE];
        
This creates a square of SIZE and attaches it to our rigid body. The box convenience constructor simply centers the box on the center of gravity of the rigid body which is always at body.pos. The center of gravity is also the point that the rigid body rotates around.

Like ChipmunkBody objects, ChipmunkShape objects have a number of properties that you might want to set:

        shape.elasticity = 0.3f;
        shape.friction = 0.3f;
    
Elasticity controls how bouncy a body is. When two shapes collide, their elasticity values are multiplied together to get the elasticity of the collision. 0 means no bounce, and 1 would make something bounce perfectly. While you can go above 1.0, it means that the object will speed up every time it hits something and will probably fly out of control. Friction works the same way, by multiplying the friction values of the two shapes to determine the final friction to apply. You can think of the friction value as being in terms of surface slopes. If the friction value is 1 then the objects will be able to stop on a 45 degree slope. If the friction value was 0.5, then the objects would be able to stop on a slope of Atan(0.5) = 26.6 degrees

Next we'll set up the properties used in collision callbacks:

        shape.collisionType = [FallingButton class];
        shape.data = self;
    
If you remember from earlier, collisionType is used as a key value when setting up collision handler callbacks and the data property was how we got a reference back to our falling button object.

Lastly, we need to implement the ChipmunkObject protocol so that we don't have to add the collision shape and rigid body to the space by hand. To do that, all we need to do is implement a single method chipmunkObjects which returns an NSSet that contains all the basic Chipmunk objects this game object uses. The easiest way to do that is to use a synthesized property. Then all you have to do is create an instance variable named chipmunkObjects and use the ChipmunkObjectFlatten() function to initialize it. You can pass anything that implements the ChipmunkObject protocol to ChipmunkObjectFlatten(). Just don't forget about the nil terminator and don't forget to retain the set it returns!

        chipmunkObjects = [ChipmunkObjectFlatten(body, shape, nil) retain];
        
Putting this all together, our initialization method looks like this:

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
        
#Button Action:

The only thing remaining is to make the method that gets called when the button is clicked.
static cpFloat frand_unit(){return 2.0f*((cpFloat)rand()/(cpFloat)RAND_MAX) - 1.0f;}

        - (void)buttonClicked {
          cpVect v = cpvmult(cpv(frand_unit(), frand_unit()), 300.0f);
          body.vel = cpvadd(body.vel, v);
          
          body.angVel += 5.0f*frand_unit();
        }
Nothing too complicated there. We just add a random change to the body's velocity and another random change to it's angular velocity.

#Closing Thoughts:

Now that you've seen how easy and powerful Objective-Chipmunk can be, why not integrate it into your iPhone game? The Objective-C API will save you time (and money) by providing you a high level API that works well with popular iPhone libraries such as Cocos2D. You'll also spend less time worrying about memory management as it allows you to manage your Chipmunk memory just like the rest of your iPhone app.

Happy Chipmunking!

