## Function order
Boy, we covered a LOT in the last section! We saw how we could translate, rotate and scale an object to create the world matrix. We also saw how we can use lookAt to create a view matrix.

Now, i always say matrix multiplication is assisiative, but not cumulative. This means:

* A \* B \* C = A \* (B \* C)
* A \* B != B \* A

So, whats the right order to multiply matrices in? Well, with the __modelView__ matrix you would expect to multiply the model by the view matrix, but that is NOT the case. That's right, we're about to make matrices a LOT more confusing. But first, let's take a look at the full multiplication that's happening.

Remember, OpenGL is column major, so vectors are column matrices (1 x 4). Because matrix multiplication inner products must match, this means the vector goes on the right side. The entire transform pipeline looks like this:

```
In theory
translated vertex = vertex * model * view * projection
How OpenGL really does it
renderVertex = vertex * modelView * projection
```

We want to first transform into model space, then view space, then projection space. Column major matrices have a slight gotcha to them, the matrices take effect from left to right, not right to left. 

This means that the above code, would first move into projection space, then view space then model space. The exact Opposite of what we want!

This is confusing, but in order to do this:

```
translated vertex = vertex -> model -> view -> projection
```

You have to multiply in reverse order, like so:

```
translated vertex = projection * view * model * vertex
```

This is not the case for row major matrices! Only column major ones.

## Scaling, Translating and Rotating
The other order we need to worry about is scale-translate-rotate order. 

What order should you scale/translate/rotate in?

* Scale First
* Rotate second
* Translate last

So the transformation pipe looks like this:

```
transformed = vertex -> scale -> rotate -> translate
```

Remember, multiply right to leftThis means your multiplication order will look like this:

```
Result = Translate * Rotate * Scale * Vertex
```
Fun times.

## OpenGL functions
Hopefully that all made sense, so how does that apply to OpenGL functions? Well if you recall, openGL functions just multiply matrices, so we have to call them in the right order, like so

```cs
void Render() {
    // Pipeline: translated vertex = vertex -> model -> view -> projection
    // Model matrix: transformed = vertex -> scale -> rotate -> translate
    // Remember, read right to left when implementing multiplication!
    
    // TODO: Set up projection here
    // Multiply projection
    
    // Set the modelView matrix to identity
    GL.MatrixMode(MatrixMode.ModelView);
    GL.LoadIdentity();
    
    // Multiply view
    LookAt(0.5f, 0.5f, 0.5f, 0.0f, 0.0f, 0.0f, 0.0f, 1.0f, 0.0f);

    // Draw an object
    // Multiply model (scale, rotate, translate)
    // Multiply translate
    GL.Translate(-3.0f, 0.0f, 1.0f);
    // Multiply rotate
    GL.Rotate(90.0f, 1.0f, 0.0f, 0.0f);
    // Multiply scale
    GL.Scale(1.0f, 1.0f, 1.0f); // No scale
    
    // The actual model has the input vertices
    DrawCube();
}
```

If this doesn't make sense right now, go ahead and give me a call on skype. I know it's a bit of a confusing topic.

# Box Demo
Let's make a demo that draws a cube in the scene. This cube will have some rotation, scale and translation for you to play around with.

First, make a new demo scene and follow along

```
using System;
using OpenTK.Graphics.OpenGL;

namespace GameApplication {
    class BoxDemo : Game {
        Grid grid = null;

        // Add Look at function, it's in this gist
        // https://gist.github.com/gszauer/91038dbb010755d719de

        public override void Initialize() {
            grid = new Grid();
        }

        public override void Update(float dTime) {

        }
        public override void Render() {
            GL.MatrixMode(MatrixMode.Modelview);
            GL.LoadIdentity(); // Reset modelview matrix
            LookAt(
                0.5f, 0.5f, 0.5f, // Position
                0.0f, 0.0f, 0.0f, // Target
                0.0f, 1.0f, 0.0f  // Up
            );
    
            // Render grid at the origin of the world
            grid.Render();
        }
        
        public override void Shutdown() {

        }
    }
}
```

This puts us exactly where we left off in the last section. With the grid being visible at an angle. Remember, your machine will not look 100% like mine. We should have the same image, but yours might be culled out on the side.

Next, let's add a function that draws a cube. How do you draw a cube? A lot of triangles. I went ahead and drew one on paper, then counted all the verts out into CCW triangles. The result is [this static function](https://gist.github.com/gszauer/9cee93bbe25b7fd9f5da). 

Go ahead and add the above function to your class. In render, after drawing the grid set the render color to red, and draw the cube. Like so:

```
 public override void Render() {
    GL.MatrixMode(MatrixMode.Modelview);
    GL.LoadIdentity(); // Reset modelview matrix
    LookAt(
        0.5f, 0.5f, 0.5f, // Position
        0.0f, 0.0f, 0.0f, // Target
        0.0f, 1.0f, 0.0f  // Up
    );

    // Render grid at the origin of the world
    grid.Render();

    // Set render color
    GL.Color3(1.0f, 0.0f, 0.0f);
    // Draw cube
    DrawCube();
}
```

When you run the game one of two things is going to happen. Either the entire screen is going to be red, or you will see the scene as before, with no changes. This is because the size of the cube is -1 to +1. The same size as NDC. The cube will either fill NDC and render the whole screen red, or get clipped and not render at all.Depends on your graphics card's default Z-Test setting.

## On your own
We now have a rudimentary 3D scene! I want you to apply the following transformation to the 3D cube on your own:

* Scale to 0.05f on all axis
* Rotate 73 degrees on the Y axis
* Rotate 45 degrees on the X axis
* Translate to 0.25f on the X axis 
* Translate to -0.25f on the Z axis

Keep in mind, if your pipeline is

```
scale -> rotate -> translate
```

Then the multiplication order is 

```
translate * scale * rotate
```

I'll give you a bit of a hint as to where to do this. The full transform pipeline is this:

```
render vertex = projection -> model -> view -> scale -> rotate -> translate -> original vertex
```

We don't have projection yet. No action there

So you want to set the modelview matrix with the lookAt function (this was already done in the previous step, it does not change)

Next you want to render the grid at origin. We don't want to apply any transformations to the grid. You could say, the grid is only transformed by the view matrix, it has no model matrix.

Next you want to do your Scaling, Rotating and Translating. This is going to be the "model" part of the _modelView_ matrix. It will position the square in the scene.

Last, you simply set the color to red (already there from previous section) and render by calling DrawCube (already there from previous section).

The resulting screen will look kind of like this:

![SPACE](c_space.png)

If you get stuck, this page has a subsection with the solution. If i'm on skype, message me before you look at the solution, we can work trough the problem.