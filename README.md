# Android OpenGLES2.0 编程教程系列
本文解释了怎样在一个简单的应用Activity中完成GLSurfaceView 和 GLSurfaceView.Renderer的极简实现。
原文地址：https://developer.android.com/training/graphics/opengl/environment.html

翻译地址：http://blog.csdn.net/u011520181/article/details/51525766
### 在注册文件中声明OpenGL ES使用

```
<uses-feature android:glEsVersion="0x00020000" android:required="true" />  
```
如果你的应用使用纹理压缩，那么你也必须声明你的应用支持那种格式的压缩，以便于它只会被安装在兼容的设备上。

```
<supports-gl-texture android:name="GL_OES_compressed_ETC1_RGB8_texture" />  
<supports-gl-texture android:name="GL_OES_compressed_paletted_texture" />  
```
### 为OpenGL ES图形创建Activity，使用GLSurfaceView 作为主视图的一个Activity的极简实现
GLSurfaceView中其实不需要做太多工作，实际的绘制任务都在GLSurfaceView.Renderer中了。所以GLSurfaceView中的代码也非常少，你可以直接使用GLSurfaceView

```
public class OpenGLES20Activity extends AppCompatActivity {
    private GLSurfaceView mGLView;
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mGLView = new GLSurfaceView(this);
        //创建一个OpenGL ES 2.0 context
        mGLView.setEGLContextClientVersion(2);
        //设置Renderer到GLSurfaceView
        mGLView.setRenderer(new MyGLRenderer());
        // 只有在绘制数据改变时才绘制view
        mGLView.setRenderMode(GLSurfaceView.RENDERMODE_WHEN_DIRTY);
        setContentView(mGLView);
    }
}
```
### 构建一个Renderer类
在一个使用OpenGL
 ES的应用中实现GLSurfaceView.Renderer类或渲染器，事情将开始变得有趣。这个类将控制用它关联的内容在GLSurfaceView上绘制什么。渲染器中有三个被Android系统调用的用来计算在GLSurfaceView上画什么以及怎么画的方法。
     * onSurfaceCreated()，调用一次用来设置view的OpenGL ES环境；
     * onDrawFrame()，调用用来重绘view；
     * onSurfaceChanged()，当view的几何改变时调用，比如当设备屏幕的方向发生改变时；
这里有一个非常基础的OpenGL ES渲染器的实现，它除了在GLSurfaceView上绘制一个黑色背景什么也不做：

```
public class MyGLRenderer implements GLSurfaceView.Renderer {  
  
    public void onSurfaceCreated(GL10 unused, EGLConfig config) {  
        //设置背景的颜色 
        GLES20.glClearColor(0.0f, 0.0f, 0.0f, 1.0f);  
    }  
  
    public void onDrawFrame(GL10 unused) {  
        // 重绘背景色
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);  
    }  
  
    public void onSurfaceChanged(GL10 unused, int width, int height) {  
        GLES20.glViewport(0, 0, width, height);  
    }  
}  
```
以上就是所有需要做的东西！上面的代码们创建了一个简单的Android应用，它使用OpenGL显示了一个灰色的屏幕。但这段代码并没有做什么有趣的事情，只是为使用OpenGL绘图做好了准备。

注:你可以不明白为什么这些方法们都具有一个GL10参数，但你使用的却是OpengGLES 2.0 API们。这其实是为了使Android框架能简单的兼容各OpenGLES版本而做的。

### 定义形状
会定义在OpenGLES view上所绘制的形状，是你创建高端图形应用杰作的第一步。如果你不懂OpenGLES定义图形对象的一些基本知识，使用OpenGLES可能有一点棘手。

本文解释OpenGLES相对于Android设备屏幕的坐标系统、定义一个形状的基础知识、形状的外观、以及如何定义三角形和正方形。
OpenGLEs允许你使用坐本在三个维度上定义绘制对象。所以，在你可以绘制一个三角形之前，你必须定义它的坐标。在OpenGL中，典型的方式是为坐标定义一个浮点类型的顶点数组。为了最高效，你应把这些坐标都写进一个ByteBuffer，它会被传到OpenGLES图形管线以进行处理。

```
class Triangle {

    private FloatBuffer vertexBuffer;

    // 数组中每个顶点的坐标数
    static final int COORDS_PER_VERTEX = 3;
    static float triangleCoords[] = { // 按逆时针方向顺序:
         0.0f,  0.622008459f, 0.0f,   // top
        -0.5f, -0.311004243f, 0.0f,   // bottom left
         0.5f, -0.311004243f, 0.0f    // bottom right
    };

    // 设置颜色，分别为red, green, blue 和alpha (opacity)
    float color[] = { 0.63671875f, 0.76953125f, 0.22265625f, 1.0f };

    public Triangle() {
        // 为存放形状的坐标，初始化顶点字节缓冲
        ByteBuffer bb = ByteBuffer.allocateDirect(
                // (坐标数 * 4)float占四字节
                triangleCoords.length * 4);
        // 设用设备的本点字节序
        bb.order(ByteOrder.nativeOrder());

        // 从ByteBuffer创建一个浮点缓冲
        vertexBuffer = bb.asFloatBuffer();
        // 把坐标们加入FloatBuffer中
        vertexBuffer.put(triangleCoords);
        // 设置buffer，从第一个坐标开始读
        vertexBuffer.position(0);
    }
}
```
缺省情况下，OpenGLES 假定[0,0,0](X,Y,Z) 是GLSurfaceView 帧的中心，[1,1,0]是右上角，[-1,-1,0]是左下角。

注意此形状的坐标是按逆时针方向定义的。绘制顺序很重要，因为它定义了哪面是形状的正面，哪面是反面，使用OpenGLES 的cullface特性，你可以只画正面而不画反面。

### 定义一个正方形
在OpenGL中定义正方形是十分容易的，有很多方法能做的，但是典型的做法是使用两个三角形：
![ccw-square](http://ofoygdb8c.bkt.clouddn.com/ccw-square.png)
你要为两个三角形都按逆时针方向定义顶点们，并且将这些坐标值们放入一个ByteBuffer中。为了避免分别为两个三角形定义两个坐标数组，我们使用一个绘制列表来告诉OpenGLES图形管线如果画这些顶点们。下面就是这个形状的代码：

```
class Square {

    private FloatBuffer vertexBuffer;
    private ShortBuffer drawListBuffer;

    // 每个顶点的坐标数
    static final int COORDS_PER_VERTEX = 3;
    static float squareCoords[] = { -0.5f,  0.5f, 0.0f,   // top left
                                    -0.5f, -0.5f, 0.0f,   // bottom left
                                     0.5f, -0.5f, 0.0f,   // bottom right
                                     0.5f,  0.5f, 0.0f }; // top right

    private short drawOrder[] = { 0, 1, 2, 0, 2, 3 }; // 顶点的绘制顺序

    public Square() {
        // initialize vertex byte buffer for shape coordinates
        ByteBuffer bb = ByteBuffer.allocateDirect(
        // (坐标数 * 4)
                squareCoords.length * 4);
        bb.order(ByteOrder.nativeOrder());
        vertexBuffer = bb.asFloatBuffer();
        vertexBuffer.put(squareCoords);
        vertexBuffer.position(0);

        // 为绘制列表初始化字节缓冲
        ByteBuffer dlb = ByteBuffer.allocateDirect(
        // (对应顺序的坐标数 * 2)short是2字节
                drawOrder.length * 2);
        dlb.order(ByteOrder.nativeOrder());
        drawListBuffer = dlb.asShortBuffer();
        drawListBuffer.put(drawOrder);
        drawListBuffer.position(0);
    }
}
```

### 绘制形状
你定义了要绘制的形状后，你就要画它们了。使用OpenGLES 2.0会形状会有一点点复杂，因为API提供了大量的对渲染管线的控制能力。
在你做任何绘制之前，你必须初始化形状然后加载它。除非形状的结构(指原始的坐标们)在执行过程中发生改变，你都应该在你的Renderer的方法onSurfaceCreated()中进行内存和效率方面的初始化工作。

```
public void onSurfaceCreated(GL10 unused, EGLConfig config) {
    ...

    // 初始化一个三角形
    mTriangle = new Triangle();
    // 初始化一个正方形
    mSquare = new Square();
}
```

使用OpenGLES 2.0画一个定义好的形状需要一大坨代码，因为你必须为图形渲染管线提供一大堆信息。典型的，你必须定义以下几个东西：

VertexShader-用于渲染形状的顶点的OpenGLES 图形代码。
FragmentShader-用于渲染形状的外观（颜色或纹理）的OpenGLES 代码。
Program-一个OpenGLES对象，包含了你想要用来绘制一个或多个形状的shader。
你至少需要一个vertexshader来绘制一个形状和一个fragmentshader来为形状上色。这些形状必须被编译然后被添加到一个OpenGLES program中，program之后被用来绘制形状。下面是一个展示如何定义一个可以用来绘制形状的基本shader的例子：

```
    private final String vertexShaderCode = //渲染形状的顶点的OpenGLES 图形代码
            "uniform mat4 uMVPMatrix;" +
                    "attribute vec4 vPosition;" +
                    "void main() {" +
                    "  gl_Position = uMVPMatrix * vPosition;" +
            "}";

    private final String fragmentShaderCode = //渲染形状的外观（颜色或纹理）的OpenGLES 代码。
            "precision mediump float;" +
                    "uniform vec4 vColor;" +
                    "void main() {" +
                    "  gl_FragColor = vColor;" +
             "}";
```

Shader们包含了OpenGLShading Language (GLSL)代码，必须在使用前编译。要编译这些代码，在你的Renderer类中创建一个工具类方法：

```
public static int loadShader(int type, String shaderCode){

    // 创建一个vertex shader类型(GLES20.GL_VERTEX_SHADER)
    // 或fragment shader类型(GLES20.GL_FRAGMENT_SHADER)
    int shader = GLES20.glCreateShader(type);

    // 将源码添加到shader并编译之
    GLES20.glShaderSource(shader, shaderCode);
    GLES20.glCompileShader(shader);

    return shader;
}
```

为了绘制你的形状，你必须编译shader代码，添加它们到一个OpenGLES program 对象然后链接这个program。在renderer对象的构造器中做这些事情，从而只需做一次即可。

注:编译OpenGLES shader们和链接linkingprogram们是很耗CPU的，所以你应该避免多次做这些事。如果在运行时你不知道shader的内容，你应该只创建一次code然后缓存它们以避免多次创建。

```
public Triangle() {
    ...
    int vertexShader = loadShader(GLES20.GL_VERTEX_SHADER, vertexShaderCode);
    int fragmentShader = loadShader(GLES20.GL_FRAGMENT_SHADER, fragmentShaderCode);

    // 创建一个空的OpenGL ES Program
    mProgram = GLES20.glCreateProgram();
    // 将vertex shader添加到program
    GLES20.glAttachShader(mProgram, vertexShader);
    // 将fragment shader添加到program
    GLES20.glAttachShader(mProgram, fragmentShader);
    // 创建可执行的 OpenGL ES program
    GLES20.glLinkProgram(mProgram);          
}
```

此时，你已经准备好增加真正的绘制调用了。需要为渲染管线指定很多参数来告诉它你想画什么以及如何画。因为绘制操作因形状而异，让你的形状类包含自己的绘制逻辑是个很好主意。

创建一个draw()方法负责绘制形状。下面的代码设置位置和颜色值到形状的vertexshader和fragmentshader，然后执行绘制功能。

```
public void draw() {
    // 将program加入OpenGL ES环境中
    GLES20.glUseProgram(mProgram);

    // 获取指向vertex shader的成员vPosition的 handle
    mPositionHandle = GLES20.glGetAttribLocation(mProgram, "vPosition");

    // 启用一个指向三角形的顶点数组的handle
    GLES20.glEnableVertexAttribArray(mPositionHandle);

    // 准备三角形的坐标数据
    GLES20.glVertexAttribPointer(mPositionHandle, COORDS_PER_VERTEX,
                                 GLES20.GL_FLOAT, false,
                                 vertexStride, vertexBuffer);

    // 获取指向fragment shader的成员vColor的handle 
    mColorHandle = GLES20.glGetUniformLocation(mProgram, "vColor");

    // 设置三角形的颜色
    GLES20.glUniform4fv(mColorHandle, 1, color, 0);

    // 画三角形
    GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, vertexCount);

    // 禁用指向三角形的顶点数组
    GLES20.glDisableVertexAttribArray(mPositionHandle);
}
```

一旦你完成这些代码，画这个对象只需要在Renderer的onDrawFrame()调用draw()方法。

### 投影和相机视图的应用
投影变化的数据是在你GLSurfaceView.Renderer类的onSurfaceChanged()方法中被计算的。下面的示例代码是获取GLSurfaceView的高和宽，并通过Matrix.frustumM()方法用它们填充到投影变换矩阵中。

```
// mMVPMatrix is an abbreviation for "Model View Projection Matrix"  
private final float[] mMVPMatrix = new float[16];  
private final float[] mProjectionMatrix = new float[16];  
private final float[] mViewMatrix = new float[16];  
  
@Override  
public void onSurfaceChanged(GL10 unused, int width, int height) {  
    GLES20.glViewport(0, 0, width, height);  
  
    float ratio = (float) width / height;  
  
    // this projection matrix is applied to object coordinates  
    // in the onDrawFrame() method  
    Matrix.frustumM(mProjectionMatrix, 0, -ratio, ratio, -1, 1, 3, 7);  
}  
```

上面的代码填充有一个投影矩阵mProjectionMatrix，mProjectionMatrix可以在onFrameDraw()方法中与下一部分的相机视图结合在一起。
注意：如果仅仅只把投影矩阵应用的到你绘制的对象中，通常你只会得到一个非常空的显示。一般情况下，你还必须为你要在屏幕上显示的任何内容应用相机视图。
通过在你的渲染器中添加相机视图变换作为你绘制过程的一部分来完成你的绘制图像的变换过程。在下面的代码中，通过Matrix.setLookAtM()方法计算相机视图变换，然后将其与之前计算出的投影矩阵结合到一起。合并后的矩阵接下来会传递给绘制的图形。

```
@Override  
public void onDrawFrame(GL10 unused) {  
    ...  
    // 设置相机的位置
    Matrix.setLookAtM(mViewMatrix, 0, 0, 0, -3, 0f, 0f, 0f, 0f, 1.0f, 0.0f);  
  
    // 计算投影和相机的转换 
    Matrix.multiplyMM(mMVPMatrix, 0, mProjectionMatrix, 0, mViewMatrix, 0);  
  
    // Draw shape  
    mTriangle.draw(mMVPMatrix);  
}  
```

修改你的图形对象的draw()方法来接收联合变换矩阵，并将它们应用到图形中：

```
public void draw(float[] mvpMatrix) { // pass in the calculated transformation matrix  
    ...  
  
    // get handle to shape's transformation matrix  
    mMVPMatrixHandle = GLES20.glGetUniformLocation(mProgram, "uMVPMatrix");  
  
    // Pass the projection and view transformation to the shader  
    GLES20.glUniformMatrix4fv(mMVPMatrixHandle, 1, false, mvpMatrix, 0);  
  
    // Draw the triangle  
    GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, vertexCount);  
  
    // Disable vertex array  
    GLES20.glDisableVertexAttribArray(mPositionHandle);  
}  
```

### 旋转一个图形
用OpenGL ES 2.0来旋转一个绘制对象是相对简单的。在你的渲染器中，添加一个新的变换矩阵（旋转矩阵），然后把它与你的投影与相机视图变换矩阵合并到一起：

```
private float[] mRotationMatrix = new float[16];  
public void onDrawFrame(GL10 gl) {  
    float[] scratch = new float[16];  
  
    ...  
  
    // Create a rotation transformation for the triangle  
    long time = SystemClock.uptimeMillis() % 4000L;  
    float angle = 0.090f * ((int) time);  
    Matrix.setRotateM(mRotationMatrix, 0, angle, 0, 0, -1.0f);  
  
    // Combine the rotation matrix with the projection and camera view  
    // Note that the mMVPMatrix factor *must be first* in order  
    // for the matrix multiplication product to be correct.  
    Matrix.multiplyMM(scratch, 0, mMVPMatrix, 0, mRotationMatrix, 0);  
  
    // Draw triangle  
    mTriangle.draw(scratch);  
}  
```

### 设置触摸事件
为了你的OpenGL ES应用能够响应触摸事件，你必须在你的GLSurfaceView中实现onTouchEvent()方法，下面的实现例子展示了怎样监听MotionEvent.ACTION_MOVE事件，并将该事件转换成图形的旋转角度。

```
private final float TOUCH_SCALE_FACTOR = 180.0f / 320;  
private float mPreviousX;  
private float mPreviousY;  
  
@Override  
public boolean onTouchEvent(MotionEvent e) {  
    // MotionEvent reports input details from the touch screen  
    // and other input controls. In this case, you are only  
    // interested in events where the touch position changed.  
  
    float x = e.getX();  
    float y = e.getY();  
  
    switch (e.getAction()) {  
        case MotionEvent.ACTION_MOVE:  
  
            float dx = x - mPreviousX;  
            float dy = y - mPreviousY;  
  
            // reverse direction of rotation above the mid-line  
            if (y > getHeight() / 2) {  
              dx = dx * -1 ;  
            }  
  
            // reverse direction of rotation to left of the mid-line  
            if (x < getWidth() / 2) {  
              dy = dy * -1 ;  
            }  
  
            mRenderer.setAngle(  
                    mRenderer.getAngle() +  
                    ((dx + dy) * TOUCH_SCALE_FACTOR));  
            requestRender();  
    }  
  
    mPreviousX = x;  
    mPreviousY = y;  
    return true;  
}  
```

需要注意的是，计算完旋转角度后，需要调用requestRender()方法来告诉渲染器是时候渲染帧画面了。
当渲染代码是在独立于你应用程序的主用户界面线程的单独线程执行的时候，你必须声明这个共有变量是volatile类型的。下面的代码声明了这个变量并且暴露了它的getter和setter方法对：

```
    ...  
    
    public volatile float mAngle;  
  
    public float getAngle() {  
        return mAngle;  
    }  
  
    public void setAngle(float angle) {  
        mAngle = angle;  
    }  
}  
```

为了应用触摸输入产生的旋转，先注释掉产生角度的代码，并添加一个右触摸事件产生的角度mAngle：

```
public void onDrawFrame(GL10 gl) {  
    ...  
    float[] scratch = new float[16];  
  
    // Create a rotation for the triangle  
    // long time = SystemClock.uptimeMillis() % 4000L;  
    // float angle = 0.090f * ((int) time);  
    Matrix.setRotateM(mRotationMatrix, 0, mAngle, 0, 0, -1.0f);  
  
    // Combine the rotation matrix with the projection and camera view  
    // Note that the mMVPMatrix factor *must be first* in order  
    // for the matrix multiplication product to be correct.  
    Matrix.multiplyMM(scratch, 0, mMVPMatrix, 0, mRotationMatrix, 0);  
  
    // Draw triangle  
    mTriangle.draw(scratch);  
}  
```


