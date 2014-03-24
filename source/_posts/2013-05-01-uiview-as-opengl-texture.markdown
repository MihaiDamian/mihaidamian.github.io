---
layout: post
status: publish
published: true
title: Using UIViews as OpenGL textures for custom view controller transitions
categories:
- iOS
tags:
- UIKit
- iOS
- OpenGL
- texture
comments: []
---
In [a previous post][{% post_url 2013-02-05-physics-uikit-2 %}] I've demonstrated how UIKit can be used with a physics engine. For this post I'm going to continue on the topic of mixing UIKit with technologies from the field of game development and show how UIViews can be used as textures in OpenGL.

As a practical example I've put together a sample app that simulates a <em>UINavigationController</em> where view controller transitions are animated using a cube instead of the standard slide animations. Go ahead and download the complete <a title="source code" href="https://github.com/MihaiDamian/Cube-transition-example" target="_blank">source code</a>. You can see it in action in the video below:

{% youtube ZOpnpLiykE4 %}

Before we go any further I feel a disclaimer about the choice to use OpenGL in this example is needed. I have to admit it is entirely possible to construct simple 3D animations like this one using nothing but Core Animation - which being a higher level technology is easier to use. In fact <em>CATransition</em> already implements an undocumented "cube" animation identical with what I've built here from scratch. The only purpose of this example is to present the steps and challenges you need to go through in order to transform UIViews into OpenGL textures.
<h3>Drawing the view to an image</h3>
The function that makes this entire exercise possible is <em>CALayer</em>'s <em>renderInContext:</em> method. As the name suggests, this will draw a <em>CALayer</em> into a <em>CGContext</em>. By drawing into a <em>CGBitmapContext</em> (which is a subtype of <em>CGContext</em>). We can then grab the image data and pass it to OpenGL using the <em>glTexImage2D</em> function. One point to remember here is that neither Core Graphics nor OpenGL know anything about points - they only work in pixels. So whenever we transfer UIKit sizes and positions into Core Graphics or OpenGL we always need to take into account the content scale factor.

You'll find the code implementing all this in the <em>TextureAtlas</em> class. Notice the code there actually draws two views in a single texture. The input views will be the views of the view controllers involved in the transition. The reason why we render the views side by side in the same texture is that shaders (in our case thinly wrapped by <em>GLKBaseEffect</em>) normally draw a single texture in a draw call. In our case this trick simplifies the code a bit, but when rendering more complex scenes it also helps to improve performance if you manage to group up your drawing needs into fewer draw calls. This is because sending data to the graphics pipeline is usually the performance bottleneck and not the rendering itself which runs on optimized hardware.

Second think I want to point out here is the commented code I left in <em>TextureAtlas</em>. If you uncomment it you will see the texture saved as an image file in the application's Documents folder. If you open up the file you will see it's flipped on the y-axis. The happens because by default <em>CGContext</em>'s coordinate system defines the origin point to be in the lower-left corner. That may come as some surprise if you used <em>CGContext</em> in other places like in <em>UIView'</em>s <em>drawRect:</em> where the y-axis is actually flipped for your convenience to match the coordinate system of UIKit. But as OpenGL uses the same coordinate system as Core Graphics, no extra handling is needed in our case.
<h3>The right time to rasterize the views</h3>
Let's take a look at the structure of the sample app. The root view controller is <em>NavigationController</em>, a custom view controller container that mimics the native <em>UINavigationController</em>. We have <em>AnimationViewController</em>, a wrapper for the transition animation that takes two views as parameters to kick off the animation. Also there are two dummy view controllers: <em>FirstViewController</em> and <em>SecondViewController</em>.

<em>NavigationController</em> is initialized with an instance of <em>FirstViewController</em>, which immediately gets added as a child view controller to the navigation controller. Now let's say we need to present an instance of <em>SecondViewController</em>, using the <em>pushViewController:</em> method. What the navigation controller should do here is add the <em>SecondViewController</em> as a child so we can grab it's view for the animation, add an <em>AnimationViewController</em> as a child, remove the <em>FirstViewController</em>, wait for the animation to complete and finally remove the <em>AnimationViewController</em>.

This seems straightforward but there is one thing we need to take care of. Once the animation starts, the views from the animated controllers are rasterized to textures and no future updates to them will be visible until the animation is done. View layouts (either triggered by autolayout or autoresize masks) are performed on the main thread but asynchronously from view initialization. If we simply grab the view from the <em>SecondViewController</em> as soon as it's instantiated, we might end up with a view that's the wrong size. Instead we can use <em>UIViewController</em>'s <em>transitionFromViewController:toViewController:duration:options:animations:completion:</em> method to get us out of this problem. We can leverage the fact that view layout is done by the time this method's animation block parameter gets called.
<h3>Putting it all together</h3>
Once we have the textures ready, what's left is the fairly standard task of creating an OpenGL scene for the animation. Namely we'll need to create a polygon mesh for a cube (complete with texture coordinates and surface normals for lighting), rotate the cube and call a delegate when the cube has completed 90 degrees of rotation. You can have a look at how this is implemented in the <em>Cube</em> and <em>AnimationViewController</em> classes. GLKit really helps simplify the effort here. It can take care of iOS specific things like setting up a UIView for OpenGL rendering, setting up an animation loop tied to the refresh rate of the device's screen or pausing the animation loop when the app goes into background. You can also use it for out of the box shaders and for linear algebra tasks common in 3D applications.
