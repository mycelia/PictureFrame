<h1 align="center">PictureFrame 3.2.0</h1>

<p align="center">Camera/view/GUI/window/application surface calculator for GameMaker 2024.6</p>

<p align="center"><a href="https://github.com/JujuAdams/PictureFrame/releases/">Download the .yymps</a></p>

&nbsp;

# Introduction

Getting your game's scaling right in GameMaker is an eternal chore. There are multiple scaling factors applied throughout the render pipeline which are all multiplied together to give you the final image displayed to players. This is hard enough to wrap your head around for simple games. If you're looking to add an extra bit of polish by adapting to different screen resolutions and aspect ratios then organising the various render stages becomes very confusing very fast.

PictureFrame will calculate the following output values for you:
- Camera width/height
- Viewport width/height
- Application surface width/height
- GUI layer width/height
- Application surface draw position and draw scale
- Size of black bar margins around the application surface when the application surface is small than the window

You can then apply these values yourself, or call `PfApply()` to have PictureFrame take care of it for you. Alternatively, you can call `PfCalculate()` and manage the pipeline yourself using the output.

PictureFrame uses the following input constraints:
- Camera minimum width/height (the "safe area")
- Camera maximum width/height
- Camera overscan size (useful for smoothing pixel perfect camera movement)
- Maximum scale for the viewport relative to the camera dimensions
- Whether the viewport should be pixel perfect (a.k.a. an integer scale of the camera)
- Whether the game should be displayed fullscreen
- Size of the game window
- The target width and/or target height for the GUI layer (or neither)
- Whether the application surface should be drawn as "pixel perfect" (a.k.a. at an integer scale)
- Overscan scale for adapting to CRT monitors (a compliance requirement for PS4 games)

PictureFrame is suitable for pixel art games or for high resolution games. It can calculate correct camera sizes regardless of device and is especially suited for the wide variety of aspect ratios found on mobile devices. PictureFrame only supports rendering of one view at a time and does not handle split-screen games.

&nbsp;

# Setting Up

*This guide assumes you'll be using `PfApply()` to set up GameMaker's rendering pipeline. If you're only using `PfCalculate()` then there are no additional considerations: just call the function and use the values as you see fit!*

PictureFrame is easy to set up, just a handful of function calls, but there are a couple essential steps that can be missed. Here's the big caveat one up front: **If you call `PfApply()` then you will need to call `PfPostDrawAppSurface()` in the Post-Draw event.** You can still apply post-processing shaders as you would otherwise when drawing the application surface.

Here are the basic setup steps:

1. At the start of the game, call one of the config struct creator functions. These are `PfConfigPixelArt()`, `PfConfigHighRes()`, and `PfConfigGeneral()`. The latter is the most flexible. All three functions will return the same type of struct, you can read more about what variables can be adjusted in the [script documentation](https://github.com/JujuAdams/PictureFrame/blob/dev/scripts/PfConfigGeneral/PfConfigGeneral.gml).

2. Call `PfApply()` using the config struct we just created. You may want to set the optional `resizeWindow` parameter to `true` the first time you call this function in the flow of your game.

3. Create a Post-Draw event in a persistent object instance and call `PfPostDrawAppSurface()` in that event.

And then run the game and cross your fingers!

&nbsp;

# Smooth Camera Movement

PictureFrame can help you make smooth camera movement even when using a pixel perfect camera. You'll often find that your camera will "jitter" at low speed due to the game needing to rendering pixels that don't neatly fit to the pixel grid. Whilst this is technically accurate it's also visually unpleasant, especially on large screens when the game is scaled up a long way.

Solving this problem manually yourself is a bit fiddly. You'll need to render some extra pixels around the edge of your camera on the application surface. GameMaker's camera position should be set as an integer but you'll need to store the camera position yourself as a decimal value. Finally, when you draw the application surface, you should offset the surface rendering by the fractional part of the camera's position. This is inconvenient at the best of times and can be hard to fit into an existing rendering pipeline.

PictureFrame can do all the maths for you:

1. Set `.cameraOverscan` to `1` when creating a PictureFrame config struct (e.g. the struct returned by `PfConfigPixelArt()`)

2. Store your own camera x/y position that is the precise decimal value. When you call `camera_set_view_pos()` to set GameMaker's internal camera position make sure to `floor()` the position so that GameMaker renders at an clean integer position

3. When you call `PfPostDrawAppSurface()` in the Post-Draw event, set the `fracCameraX` and `fracCameraY` parameters to the fractional part of your camera x/y position

You can look at the [`oDemoSmoothCamera` object](https://github.com/JujuAdams/PictureFrame/tree/dev/objects/oDemoSmoothCamera) in the repo for a practical example. Here is an abbreviated copy of the code for quick reference:

```gml
/// Create Event

//Create a new config using a template config that uses a pixel perfect camera
configStruct = PfConfigPixelArt(640, 320, 640, 320);

//We'll be taking advantage of the overscan feature for smooth camera movement
configStruct.cameraOverscan = 1;

//Apply the configuration struct to the pipeline
PfApply(configStruct, true);

//Set up some camera tracking variables
var _camera = view_get_camera(0);
cameraX = camera_get_view_x(_camera);
cameraY = camera_get_view_y(_camera);
cameraTargetX = cameraX;
cameraTargetY = cameraY;
```
```gml
/// Step Event

//Move the camera when the player clicks
var _camera = view_get_camera(0);

if (mouse_check_button_pressed(mb_left))
{
    cameraTargetX = mouse_x - camera_get_view_width(_camera)/2;
    cameraTargetY = mouse_y - camera_get_view_height(_camera)/2;
}

//Lerp towards the camera target
//This will generate decimal camera positions
cameraX = lerp(cameraX, cameraTargetX, 0.1);
cameraY = lerp(cameraY, cameraTargetY, 0.1);

//Set the GameMaker camera position to integer positions
camera_set_view_pos(_camera, floor(cameraX), floor(cameraY));
```
```gml
/// Post-Draw Event

//Draw the application surface using PictureFrame's anti-jitter feature
PfPostDrawAppSurface(undefined, undefined, undefined, frac(cameraX), frac(cameraY));
```
