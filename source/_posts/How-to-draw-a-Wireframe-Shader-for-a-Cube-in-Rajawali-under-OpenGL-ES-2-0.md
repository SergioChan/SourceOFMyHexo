title: How to draw a Wireframe Shader for a Cube in Rajawali under OpenGL ES 2.0
date: 2016-11-09 17:41:34
categories: 安卓菜鸟心得


tags: [OpenGL ES, GLSL, Android, Shader, Wireframe]
---

> PS. This will be my first blog totally written in english.



![](https://ooo.0o0.ooo/2016/11/09/58227c9f45765.png)

## Setup

> **Environment:** 
>
> GLSL ES Version 100
>
> OpenGL ES Version 2.0
>
> Android 6.0
>
> Android OpenGL ES 2.0/3.0 Engine [Rajawali](https://github.com/Rajawali/Rajawali)

## First Discuss

There are several ways to draw a wireframe for a cube in Rajawali engine. We can use 

```
cube.setDrawingMode(GLES20.GL_LINE_STRIP);
```

to tell the cube to draw in wireframe mode. But because a Cube, or we can say, every Polygons or 3D Objects in OpenGL are formed by lots of dots, lines and triangles, setting drawing mode will only show you such images like below : 

![](https://ooo.0o0.ooo/2016/11/09/5822d63584db6.png)

Also, when you set DrawingMode to GL_LINE_STRIP, your custom vertex or fragment shader won't work as usual. So you will see a cube full of lines, doesn't it look messy to you?

In order to draw wireframe using fragment shader, we can simply use following fragement shader :

```
precision mediump float;

varying vec2 vTextureCoord;
varying vec4 vColor;

void main() {
        vec4 newColor = vec4(0.0, 0.0, 0.0, 1.0);
        float x = min(vTextureCoord.s, 1.0 - vTextureCoord.s);
        float y = min(vTextureCoord.t, 1.0 - vTextureCoord.t);

        float line_width = 0.01;
        
        if (x < line_width) {
            newColor.g = 1.0;
            newColor.r = 1.0;
            newColor.b = 1.0;
        }

        if (y < line_width) {
            newColor.g = 1.0;
            newColor.r = 1.0;
            newColor.b = 1.0;
        }
        gl_FragColor = newColor;
}
```

However, there is still a problem here. While using this shader, if you make your cube scale in whichever direction, this drawing 'Line' will always scale along with the cube. Obviously it's because the 0.01 we set here is a relative value. But the `vTextureCoord` we can get here which was passed by our vertex shader is only a `vec2` vector represented a 2D coordinate on this mesh, which can only range from (0.0 , 0.0) to (1.0 , 1.0). This means `vTextureCoord` can only represent a relative coordinate, which doesn't help us at all.

So, here we have to use the help from vertex shader. In vertex shader, we could get some `attribute` variables that are somehow useful. In OpenGL ES, we have three kinds of type specifiers: **uniform**, **attribute** and **varying**. Uniform variables are used for transmitting datas from CPU to GPU, the usage is as below :

- Uniform variables are binded in your shader class in CPU and GPU so you have to subclass the Shader Base Class and override some methods here. First declare an int handle for binding location  and the variables that hold your data along with the variable name in your shader (glsl) file.

```
final private String uMyDataName = "uMyData";
private int muMyDataHandle;
public float MyData;
```

- Remember to set the default value for your data when initializing.

```
public MyShader() {
	MyData = 1.0f;
}
```

- Override setLocations method to get uniform variable location from programHandle.

```
@Override
public void setLocations(final int programHandle)
{
	super.setLocations(programHandle);
	muMyDataHandle = getUniformLocation(programHandle, uMyDataName);
}
```

- Override applyParams method to apply every change of your uniform variable, also you can call `glUniform1f` method directly to pass the value into shader.

```
@Override
public void applyParams() {
	super.applyParams();
	GLES20.glUniform1f(muMyDataHandle, MyData);
}
```

- In your shader (glsl) file, add your new variable and call this variable wherever you want.

```
uniform float uMyData;

void main () {
	// get uMyData here
}
```

- When you want to change the value of `uMyData` , just change  the value of `MyShader.MyData`.

Attribute variables can only be used in vertex shaders. In Rajawali, you can call `addAttribute` to directly add custom attribute and use `GLES20.glVertexAttribPointer` to set value for the attribute variable handler. Here we have several attribute variables as below :

- aPosition
- aTextureCoord
- aNormal
- aVertexColor

Among them, **aPosition** is the most useful variable for us, it represents the vector from (0,0,0) for current vertex in model projection. **aTextureCoord** is the source of **vTextureCoord** in fragment shader. As you can see, **vTextureCoord** has a type specifier '**varying**', it's used for transmitting variables from vertex shader to fragment shader. Here it's involved with some concept of the **pipline** procedure for OpenGL, what we need to know is that we could declare **'varying float A'** both in vertex shader and fragment shader at the same time, while write in vertex shader and read in fragment shader.

### Let's begin!

So, here, in order to solve the scaling problem mentioned above, we declare three scale value for x, y and z :

```
uniform float uScaleZ;
uniform float uScaleX;
uniform float uScaleY;

// We should get current width from CPU
uniform float uWidth;

// Pass everything above to fragment shader
varying float vScaleZ;
varying float vScaleX;
varying float vScaleY;

varying float vWidth;
```

And make the vertex scale in vertex shader by changing its position :

```
vec3 directionVec = vec3(aPosition.x,aPosition.y,aPosition.z);
vec4 timeVec = vec4(directionVec, 1.0);

// Scale each axis
timeVec.x = directionVec.x * uScaleX;
timeVec.y = directionVec.y * uScaleY;
timeVec.z = directionVec.z * uScaleZ;

vec4 newPosition = uMVPMatrix * timeVec;
gl_Position = newPosition;
vTextureCoord = aTextureCoord;
```

In fact, because we are going to draw colors on mesh in fragment shader, we should pass all the variables about scaling and position from vertex shader to fragment shader. So we have to add a `vec3` :

```
varying vec3 vTimeVec;
```

And in `main()` add :

```
vec3 tmpVec = vec3(timeVec.x,timeVec.y,timeVec.z);
vTimeVec = tmpVec;
```

### Next, what should we do in fragment shader?

First, declare everything we need to get from vertex shader :

```
varying vec2 vTextureCoord;
varying vec4 vColor;
varying vec3 vTimeVec;

varying float vScaleZ;
varying float vScaleX;
varying float vScaleY;

varying float vWidth;
```

The most important step here is that when we set width for line, fragment shader should exactly split it into two parts which are drawn on different meshes. It's easy to understand, 1/2 for left side and the other 1/2 for the right side.

```
const float GT = 1.0;
const float LT = 0.0;

void main() {
    vec4 newColor = vec4(0.0,0.0,0.0,1.0);
    float lineWidth = 0.004;

    float leftX = - (vWidth * vScaleX / 2.0) + lineWidth;
    float rightX = (vWidth * vScaleX / 2.0) - lineWidth / 2.0;
    
    float bottomY = - (vWidth * vScaleY / 2.0) + lineWidth;
    float topY = (vWidth * vScaleY / 2.0) - lineWidth / 2.0;

    float backZ = - (vWidth * vScaleZ / 2.0) + lineWidth;
    float frontZ = (vWidth * vScaleZ / 2.0) - lineWidth / 2.0;

    float rX = smoothstep(leftX,rightX,vTimeVec.x);
    float rZ = smoothstep(backZ,frontZ,vTimeVec.z);
    float rY = smoothstep(bottomY,topY,vTimeVec.y);
    
    if (
    (rX == GT && rY == GT) ||
    (rX == GT && rZ == GT) ||
    (rY == GT && rZ == GT) ||
    (rX == LT && rY == LT) ||
    (rX == LT && rZ == LT) ||
    (rY == LT && rZ == LT) ||
    (rX == GT && rY == LT) ||
    (rX == GT && rZ == LT) ||
    (rY == GT && rZ == LT) ||
    (rX == LT && rY == GT) ||
    (rX == LT && rZ == GT) ||
    (rY == LT && rZ == GT)){
    	// Set your custom line color here
         newColor.g = 0.659;
         newColor.r = 0.976;
         newColor.b = 0.192;
    }

    gl_FragColor = newColor;
}
```

### Some more tricks with scaling

![](https://ooo.0o0.ooo/2016/11/09/5822e8cb5bcc5.png)

As you can see from the image above, the 0 point for y and z axis of the model projection is in the middle of the cube. It's hard to modify a cube's anchor point here because there is no concept of anchor point here. But when we scale our cube, we may want it to scale based on one of its surfaces. How can we do that? I used method below to implement a demo for bottom-based and top-based scaling :

- In vertex shader, change the shader script here :

```
// uScaleMode :
// 1: top based
// 2: bottom based

timeVec.x = directionVec.x * uScaleX;
timeVec.y = directionVec.y * uScaleY * 2.0;
timeVec.z = directionVec.z * uScaleZ;
    
if (uScaleMode == 1) {
    if (timeVec.y < 0.0) {
        timeVec.y = 0.0;
    }
} else {
    if (timeVec.y > 0.0) {
        timeVec.y = 0.0;
    }
}
```

- In fragment shader :

```
float bottomY;
float topY;

if (vScaleMode == 1.0) {
    bottomY = lineWidth;
    topY = vWidth * vScaleY - lineWidth / 2.0;
} else {
    bottomY =  - vWidth * vScaleY + lineWidth;
    topY = - lineWidth / 2.0;
}
```

### How to rorate using MVPMatrix ?

Here is a discussion on [stackoverflow](http://stackoverflow.com/questions/15837177/using-matrix-rotate-in-opengl-es-2-0).

In Rajawali, we can also use MVPMatrix to rotate the object. Although Rajawali provides us method like `material.setMVPMatrix(mvpMatrix);` to set MVPMatrix, I found it not working correctly. The matrix cannot be correctly passed into vertex shader. So I add another uniform variable here, named it `uCMVPMatrix`.

In our custom Renderer which is a subclass of `RajawaliRenderer`,  add these variables :

```
private Matrix4 mvpMatrix;
private double[] mvpMatrixValue = new double[16];

private final double[] mModelMatrix = new double[16];
private double[] mTempMatrix = new double[16];
```

When you finished initialization of your cube, get the MVPMatrix from it directly using :

```
cube = new Cube(...);
mvpMatrix = cube.getModelViewProjectionMatrix();
mvpMatrixValue = mvpMatrix.getDoubleValues();
```

Then, in `onRender`, doing Matrix calculation to rotate the cube :

```
@Override
public void onRender(final long elapsedTime, final double deltaTime) {
	...
	Matrix.setIdentityM(mModelMatrix, 0); // initialize to identity matrix
    long time = SystemClock.uptimeMillis() % 4000L;
    float mAngle = 0.090f * ((int) time);
    Matrix.setRotateM(mModelMatrix, 0, mAngle, 0, 1.0f, 0.0);
    
    mTempMatrix = mvpMatrixValue.clone();
    Matrix.multiplyMM(mvpMatrixValue, 0, mTempMatrix, 0, mModelMatrix, 0);
    
    mvpMatrix = mvpMatrix.setAll(mvpMatrixValue);
    vertShader.CMVPMatrix = mvpMatrix.getFloatValues();
}
```

In our custom vertex shader, add a uniform handle for this `uCMVPMatrix` and send value to it :

```
private int muCMVPMatrixHandle;
public float[] CMVPMatrix;

@Override
public void setLocations(final int programHandle)
{
    super.setLocations(programHandle);
    ...
    muCMVPMatrixHandle = getUniformLocation(programHandle, uCMVPMatrixName);
}

@Override
public void applyParams() {
    super.applyParams();
    ...
    GLES20.glUniformMatrix4fv(muCMVPMatrixHandle, 1, false, CMVPMatrix, 0);
}
```

Thus, rotation using Matrix should be easy in Rajawali !
