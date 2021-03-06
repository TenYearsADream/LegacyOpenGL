# Textures for UI
Texturing 3D objects is cool, but one of the most often used places for textures is UI. In this section i'm going to walk you trough how to add ui to any scene. Then, in the on your own section you will implement some UI on top of the 3D textured test scene we've been working with so far.

## Multiple coordinates

UI relies on using multiple coordinates. After all, if you have a world with a perspective camera, you don't want your UI to be perspective. Unless some UI is in world space (Like the health bar of an RTS game) the UI should be in an orthographic projection.

We not only give the UI it's own projection matrix, we also give it it's own modelview matrix. This allows us to work easily in ui space. Generally, the code for UI will go something like this:

```cs
void Render() {
// FIRST, We render the 3D scene!
    
    // Assume the matrix mode is ModelView
    // Load world modelview matrix for the 3D scene
    Matrix4 lookAt = Matrix4.LookAt(new Vector3(-7.0f, 5.0f, -7.0f), new Vector3(0.0f, 0.0f, 0.0f), new Vector3(0.0f, 1.0f, 0.0f));
    GL.LoadMatrix(Matrix4.Transpose(lookAt).Matrix);
    
    // Render 3D scene as normal
    RenderWorld();
    
// THEN, render the UI

    // Clear only the depth buffer. This way our UI will not z-test against
    // the 3D scene, because the depth buffer will be empty.
    GL.Clear(ClearBufferMask.DepthBufferBit);
    
    // Switch to projection matrix mode, backup 3D projection, load UI projection
    GL.MatrixMode(MatrixMode.Projection); // Switch
    GL.PushMatrix(); // Backup
    GL.LoadIdentity(); // Clear
    GL.Ortho(-1, 1, -1, 1, -1, 1); // Load UI Projection
    
    // Switch back to modelview mode, backup 3D modelview, clear
    GL.MatrixMode(MatrixMode.ModelView); // Switch
    GL.PushMatrix(); // Backup
    GL.LoadIdentity(); // Clear
    
    // Render the UI
    RenderUI();
    
    // Restore world (3D) projection
    GL.MatrixMode(MatrixMode.Projection();
    GL.Pop();
    
    // Restore world (3D) modelview
    GL.MatrixMode(MatrixMode.ModelView();
    GL.Pop();
    
    // We make sure the matrix mode is modelview for the next render iteration
}
```

That's a LOT of code, but it kind of shows you how rendering the UI is almost an entireley different render path alltogether. It's like rendering an overlay. Perhaps the most important thing when rendering the UI is the call to clear the depth buffer:

```
GL.Clear(ClearBufferMask.DepthBufferBit);
```

This call clears all Z-values that have been written into the depth buffer so far. By doing this, we ensure that our UI never clips against anything in world space. This should look familiar, a similar call is used in Program.cs to clear the depth and color bufers.

After the depth buffer is cleared, we back up the world space projection and reset it to UI space, next we back up the world space view matrix and replace it with the UI view matrix. And that all there is to it, now we just render the UI.

I STRONGLY suggest breaking your scene into two render functions, ```RenderWorld``` and ```RenderUI```, and calling them like i did above. This will keep the main render function ```Render``` of your scene from getting unmaintanable.

## Positioning the UI
Right now positioning the UI is a pain in the ass. Because the screen goes from -1 to 1 we have to figure out how much space a ui element takes up, then normalize that into the screen. We might end up with UI code that looks like this:

```
GL.Begin(PrimitiveType.Quads);
    GL.TexCoord2(0, 1);
    GL.Vertex3(0.1345f, 0.3345f, 0.0f);
    
    GL.TexCoord2(1, 1);  
    GL.Vertex3(0.435f, 0.3345f, 0.0f);
    
    GL.TexCoord2(1, 0);
    GL.Vertex3(0.435f, 0.1245f, 0.0f); 
    
    GL.TexCoord2(0, 0);
    GL.Vertex3(0.1345f, 0.1245f, 0.0f); 
GL.End();
```

And that is kind of awefull! I mean, what if you need to change the screen position later, this becomes a nightmare to maintain. Worse yet, it does not account for aspect ration. Worse, worse yet, because of the aspect ratio error compounded with possible floating point error it's nearly impossible to get a pixel perfect UI.

And that's the real issue here, sometimes we NEED for UI to be pixel perfect. Other times it doesn't matter, but if you are serious about making something 2D, you need to at least have the option of being pixel perfect.

## Pixel Perfect

The key to a pixel perfect UI lies in the orthographic projection matrix. Right now we have the space map to NDC space, that is a cube ranging from -1 to 1, like so:

```
GL.Ortho(-1, 1, -1, 1, -10, 10); // Load UI Projection
```

That maps 1 unit to half the width of the screen. What if we didn't do this? Instead we could change our orthographic space to map 1 unit to 1 pixel? Can we even do this? Yes, yes we can, like so:

```cs
int screenWidth = MainGameWindow.Window.Width;
int screenHeight = MainGameWindow.Window.Height;
// Ortho: left, right, bottom, top, near, far
GL.Ortho(0, screenWidth, screenHeight, 0, -1, 1);
```

Now the left of the screen is at 0, the right is at ```screenWidth``` units. This makes our orthographic projection space map 1 to 1 with pixels, meaning if we want a quad to render in the top left, 10 pixels from the top, 15 pixels from the left 20 pixels wide and 30 pixels tall, we could just do this:

```
int left = 15; // Left is 15 pixels from edge of screen
int top = 10; // Top is 10 pixels from edge of screen
int right = 15 + 20; // 20 pixels wide, right side is left + 20
int bottom = 10 + 30; // 30 pixels tall, bottom is top + 30

GL.Begin(PrimitiveType.Quads);
    GL.TexCoord2(0, 1); 
    GL.Vertex3(left, bottom, 0.0f);
    
    GL.TexCoord2(1, 1);  
    GL.Vertex3(right, bottom, 0.0f);
    
    GL.TexCoord2(1, 0);   
    GL.Vertex3(right, top, 0.0f); 
    
    GL.TexCoord2(0, 0);    
    GL.Vertex3(left, top, 0.0f);  
GL.End();
```

And that's that. We just defined a texture to be drawn using pixel coordinates. All because the orthographic projection matrix mapped the projections NDC space 1 to 1 with our windows pixel space. 

What if you wanted that same quad to be 15 pixels from the right of the screen instead of the left?

```cs
int quadWidth = 20;
int right = screenWidth - 15; // Place right side of quad 15 pixels from width of screen
int left = right - quadWidth; // Place left side of quad 20 pixels from right side of quad

// Top and bottom like normal
// Draw like normal
```

The point is, once you are in pixel space, you can figure out how to anchor UI to different parts of a window

## Utility

I suggest writing a utility function. Something along the lines of 

```cs
void DrawTexture(int texId, Rect screenRect, Rect sourceRect, Size sourceImageSize) {
    // TODO, figure out how to render quad to screen
}
```

This way, you won't have to write the above verbose GL.Begin / GL.End every time you want to render a ui quad. Also, it will make your life super easy if your projection is pixel perfect. 

Try to write the function on your own, maybe load some UI texture to test it with. If you want you can send it to me for review before doing the "On Your Own" section. The ```Primitives``` class might be a good place for this function to live.