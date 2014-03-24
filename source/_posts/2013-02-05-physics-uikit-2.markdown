---
layout: post
status: publish
published: true
title: Using physics in a UIKit based application
categories:
- iOS
tags:
- physics
- Box2D
- UIKit
- iOS
- UIView
---
Some of the popular game engines available on iOS like <a title="cocos2d" href="http://www.cocos2d-iphone.org" target="_blank">cocos2d</a> and <a title="Unity" href="http://unity3d.com" target="_blank">Unity</a> come bundled with physics engines so oftentimes the first thought when you want to add a bit of physics to your app is that you need to use one of the fancy game engines you've been hearing so much about. In this post I'll walk you through using the <a title="Box2D" href="http://box2d.org" target="_blank">Box2D</a> physics engine without using any game engine or OpenGL.

Let's start with a simple example. Create a single view iPad application project:

![]({{ site.url }}/assets/2013-02-05-physics-uikit-2/images/singleviewapp.png)
![]({{ site.url }}/assets/2013-02-05-physics-uikit-2/images/ipadapp.png)

Next we need to setup Box2D. The easiest way to do this is via <a title="CocoaPods" href="http://cocoapods.org" target="_blank">CocoaPods</a>. CocoaPods greatly simplifies library dependency management for iOS and OS X projects. If you haven't used it before you need to run the following commands in the terminal to set it up:
``` bash
$ [sudo] gem install cocoapods
$ pod setup
```
Library dependencies are declared in a text file named Podfile which needs to be placed in the same directory as your xcodeproj file. Create the file and paste in the following:
``` ruby
platform :ios
pod 'box2d', '~> 2.3'
```
This declares you are targeting iOS and want to use the Box2D library using either version 2.3 or any other minor version up to 2.4 but not including 2.4. If you're new to CocoaPods I encourage you to read about the other <a title="dependency declaration options" href="https://github.com/CocoaPods/CocoaPods/wiki/Dependency-declaration-options" target="_blank">dependency declaration options</a> available.

Final step, point your terminal to your Xcode project's folder and run
``` bash
pod install
```
This command will download the Box2D library and in addition create an Xcode workspace that contains your initial project file and a new Pods project where all your dependencies reside. From this point on you should always use the generated workspace instead of the initial project and you'll be good to go.

Let's move on to the good stuff and lay out some core Box2D concepts we'll be using in this tutorial. Box2D simulates interaction between rigid <em>bodies</em>. Bodies have a position and a set of <em>fixtures</em>. Each fixture links a body with a <em>shape</em> and a few physical properties like mass, friction and "bounciness". All bodies are of course part of a <em>world</em> and you could even have multiple worlds if you'd so desire. There are a host of other features available in Box2D that I won't be covering in this tutorial. Have a look over the excellent Box2D <a title="Box2D manual" href="http://box2d.org/manual.pdf" target="_blank">manual</a> to see what else is available.

Now the great part about Box2D is that it's completely display agnostic. Its only job is to keep track of your bodies and ensure that everything interacts realistically. This also means it knows nothing of pixels, points or anything like that. Box2D simulates the "real world" and it's units of measurement are miles, kilograms and seconds. It's up to you to convert UIKit's point based coordinates into something Box2D can handle. There's a catch though: simulating physics on objects of arbitrary size is computationally intensive. To keep things simple Box2D is optimised to handle moving objects of sizes between 0.1 and 10 meters. So considering points and meters equal will not work out very well. Instead we'll use an arbitrary scaling factor to keep bodies within reasonable size.

Let's start by declaring a few functions for converting between points to meters and back:

``` objc
static const CGFloat kPointsToMeterRatio = 32.0;

float32 PointsToMeters(CGFloat points)
{
    return points / kPointsToMeterRatio;
}

CGFloat MetersToPoints(float32 meters)
{
    return meters * kPointsToMeterRatio;
}

b2Vec2 CGPointTob2Vec2(CGPoint point)
{
    float32 x = PointsToMeters(point.x);
    float32 y = PointsToMeters(point.y);
    return b2Vec2(x, y);
}

CGPoint b2Vec2ToCGPoint(b2Vec2 vector)
{
    CGFloat x = MetersToPoints(vector.x);
    CGFloat y = MetersToPoints(vector.y);
    return CGPointMake(x, y);
}
```

I also included here two convenience functions for converting between CGPoints and Box2D vectors and back.

Next, open up ViewController.m and replace its contents with the following:

``` objc
#import "ViewController.h"
#import "World.h"

#import <QuartzCore/QuartzCore.h>

@interface ViewController ()
{
    World *_world;
}

@end

@implementation ViewController

- (void)viewDidLoad
{
    [super viewDidLoad];

    _world = [[World alloc] initWithFrame:CGRectMake(0, 0, self.view.frame.size.width, self.view.frame.size.height)];

    [self generateCircles];
}

- (void)generateCircles
{
    CGSize viewSize = self.view.frame.size;
    CGFloat radius = 60;
    srand(time(NULL));
    for(int i = 0;i < 20;i++)
    {
        UIView *circleView = [[UIView alloc] initWithFrame:CGRectMake(rand() % (int)(viewSize.width - radius * 2) + radius,
                                                                      rand() % (int)(viewSize.height - radius * 2) + radius,
                                                                      radius, radius)];
        circleView.backgroundColor = [UIColor redColor];
        circleView.layer.cornerRadius = 30;
        [self.view addSubview:circleView];
        [_world addCircleWithView:circleView];
    }
}

@end
```

The view controller starts by initializing a world object that will handle the interaction with Box2D. The world object is initialized with a frame covering the entire screen. We then create 20 views at random positions on the screen and give them a corner radius to make them look like circles. Finally we pass them to the world object.

Now create a new class called World, do a #import &lt;box2d/Box2D.h&gt; at the top of World.m and add the following ivars to World in the implementation file (which at this point needs to be renamed to World.mm to work with Box2D which is a C++ library):

``` objc
{
    b2World *_world;
    NSMutableArray *_circles;
}
```

Don't worry right now about what these objects represent - I'll talk again about them in the next bits of code. If you never mixed Objective-C and C++ code before, remember that it's always worth isolating C++ code and imports of C++/Objective-C++ files in implementation files. This saves you the trouble of renaming .m files to .mm when you import your mixed language code and keeps your compile time as low as possible since Objective-C code compiles faster than Objective-C++ code.

Moving on, declare an initialization method <em>initWithFrame</em>:

``` objc
- (id)initWithFrame:(CGRect)frame
{
    self = [super init];
    if(self != nil)
    {
        b2Vec2 gravity(0.0f, 10.0f);
        _world = new b2World(gravity);
        _circles = [NSMutableArray array];

        [self createScreenBoundsForFrame:frame];
        [self setupAnimationLoop];
    }

    return self;
}
```

This code will setup a Box2D world object with gravity defined to be pointing upwards on the Y axis - meaning objects will appear to be falling from the top to the bottom of our screen. <em>_circle</em> is an array we'll use to keep track of all the bodies we'll be adding to our world. Next we'll create the invisible borders of our screen in <em>createScreenBoundsForFrame:</em> to keep the views from falling off from the screen and finally setup an animation loop to drive everything in <em>setupAniamtionLoop</em>. Here's the code for <em>createScreenBoundsForFrame</em>:

``` objc
- (void)createScreenBoundsForFrame:(CGRect)frame
{
    b2BodyDef screenBoundsDef;
    screenBoundsDef.position.Set(0.0f, 0.0f);
    b2Body *screenBounds = _world->CreateBody(&screenBoundsDef);

    b2Vec2 worldEdges[5];

    worldEdges[0] = b2Vec2(CGPointTob2Vec2(frame.origin));
    worldEdges[1] = b2Vec2(CGPointTob2Vec2(CGPointMake(CGRectGetMinX(frame), CGRectGetMaxY(frame))));
    worldEdges[2] = b2Vec2(CGPointTob2Vec2(CGPointMake(CGRectGetMaxX(frame), CGRectGetMaxY(frame))));
    worldEdges[3] = b2Vec2(CGPointTob2Vec2(CGPointMake(CGRectGetMaxX(frame), CGRectGetMinY(frame))));
    worldEdges[4] = b2Vec2(CGPointTob2Vec2(frame.origin));

    b2ChainShape worldShape;
    worldShape.CreateChain(worldEdges, 5);
    screenBounds->CreateFixture(&amp;worldShape, 0.0f);
}
```

Here you can see a <em>screenBounds</em> body object being created. Bodies are always instantiated using the Box2D world object that will also keep track and memory manage them. Also notice how the body is created from a body definition object. This allows multiple bodies to be instantiated using the same defining properties. Next we create a chain shape that spans the left, bottom, right and upper edge of the input frame. This is a special type of shape that doesn't have a width and is used mostly for defining boundaries. Finally the shape is bounded to the body using a fixture.

Before looking at the animation loop let's introduce the function that will map UIViews to Box2D bodies:

``` objc
- (void)addCircleWithView:(UIView*)view
{
    NSAssert(view.superview != nil, @"The view parameter is not part of a view hierarchy");

    b2BodyDef bodyDef;
    bodyDef.type = b2_dynamicBody;
    bodyDef.position = CGPointTob2Vec2(view.center);
    b2Body *circle = _world->CreateBody(&bodyDef);

    b2CircleShape shape;
    shape.m_radius = PointsToMeters(view.frame.size.width / 2);

    b2FixtureDef fixtureDef;
    fixtureDef.shape = &shape;
    fixtureDef.density = 1.0f;
    fixtureDef.friction = 0.3f;
    fixtureDef.restitution = 0.8f;

    circle->CreateFixture(&fixtureDef);
    // Associate the body with the passed in view
    circle->SetUserData((__bridge void*)view);

    [_circles addObject:[NSValue valueWithPointer:circle]];
}
```

Here we create a body with the same position as the view, we add it a circle shape that matches the size of the view and a few physical properties. Later on we'll need to know what view each body represents so we save the pointer to the view in the body's user data field.

Let's move on to setting up the animation loop:

``` objc
- (void)setupAnimationLoop
{
    CADisplayLink *displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(animationLoop:)];
    [displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSDefaultRunLoopMode];
}

- (void)animationLoop:(CADisplayLink*)sender
{
    CFTimeInterval timeDelta = sender.duration * sender.frameInterval;

    int32 velocityIterations = 6;
    int32 positionIterations = 2;

    _world->Step(timeDelta, velocityIterations, positionIterations);

    // Adjust position for all views associated with circle bodies
    for(NSValue *circleWrapper in _circles)
    {
        b2Body *circle = (b2Body*)[circleWrapper pointerValue];
        UIView *view = (__bridge UIView*)circle->GetUserData();
        view.center = b2Vec2ToCGPoint(circle->GetPosition());
    }
}
```

Animations are always computed in discreet time intervals called <em>frames</em>. Animations get more accurate as frames get shorter but since the amount of time required to compute a frame stays relatively constant you take a performance hit when animating in shorter frames. Displays however operate on a fixed frame rate. In the code above, <em>animationLoop:</em> is in charge of animating and drawing a frame on the screen. CADisplayLink is in charge of deciding when the screen is ready to draw something new and will call <em>animationLoop:</em> to do the drawing. It's entirely possible that <em>animationLoop:</em> would require more time to execute than a single screen refresh cycle, but what CADisplayLink ensures is that your animations stay as smooth as possible with the available resources and that no animation frames are being computed if the display is not ready to show them.

Let's look at <em>animationLoop:</em> in more detail. First thing it does is figure out how much time has passed since the last run and passes that info to our world object which will then proceed to update its internal state. There are two extra parameters there: <em>velocityIterations</em> and <em>positionIterations</em>. These have more to do with Box2D's inner workings. What happens is that Box2D approximates body velocity and position in multiple iterations. The more iterations you attempt, the more realistic the outcome. You can fine tune these values until you find something acceptable in terms of performance and realism.

Once Box2D does its thing, all we need to do is iterate over all circle bodies and update the position of their associated UIView to match the bodies' new positions.

Finally, we need to take care of the cleanup. The only C++ object we allocated ourselves is the world object. Everything else was allocated internally by Box2D and will be cleaned up once the world object is destroyed:

``` objc
- (void)dealloc
{
    delete _world;
}
```

That's it! If you followed all the steps you should end up with something like this:
{% youtube dxox7uz4Sas %}

You can grab the full code for this tutorial from here: <a title="https://github.com/MihaiDamian/Box2DTutorial" href="https://github.com/MihaiDamian/Box2DTutorial" target="_blank">https://github.com/MihaiDamian/Box2DTutorial</a>

So there you have it. Physics engines can be used very easily in a UIKit based app. In fact you could even go and build a game like <a title="Hundreds on iTunes" href="https://itunes.apple.com/us/app/hundreds/id493536432?mt=8" target="_blank">Hundreds</a> using nothing but UIKit and Box2D.
