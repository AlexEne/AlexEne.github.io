---
layout: post
---

Some time ago I was quite bored so I wanted to learn something new. 
Probably inspired by the [Steam Dev Days talks](https://www.youtube.com/user/SteamworksDev/videos), 
OpenGL seemed like a nice idea and started reading about it. 
While there are some [great](http://www.amazon.com/OpenGL-SuperBible-Comprehensive-Tutorial-Reference/dp/0321902947/ref=pd_bxgy_b_img_y) [books](http://www.amazon.com/OpenGL-Insights-Patrick-Cozzi/dp/1439893764) out there, 
nothing compares to doing some work yourself and figuring stuff out. 

After getting accustomed with the basics of OpenGL, I began thinking about drawing lots of particles. 
There are not enough particles in current games. 
If I would make a game it would be probably have everything made out of of particles, but let's not get off-track.

One important part of this quest are compute shaders. 
Since they use GLSL, you have access to texture buffers, storage buffers, atomic memory operations, and many other useful features. 
One advantage is that they integrate almost seamless in an existing pipeline. 
You just need to pass ```GL_COMPUTE_SHADER``` as a parameter to ```glCreateShader``` and then it go through the normal 
attach shader, compile, link steps. 
One thing to remember is that compute shaders can't be mixed in the same shader program with the other graphics shaders 
( geometry, vertex, fragment, tessellation). More about this later.

The code is available on [github](https://github.com/AlexEne/GL_Particles). 
It looks quite straight-forward to me, but I wrote it so I might not have the most objective opinion about it. 
This was done in my spare time and while I tried to make it clean and correct, most likely there are things that I did wrong. 
If you spot any mistakes please leave a comment, and I will try to find the time to fix them.

Let's set up the stage. First let me enumerate the third parties used: SDL, GLEW and GLM.

[SDL](https://www.libsdl.org/) (Simple DirectMedia Layer) is an extremely clean, simple (as the name suggests) library that handles input, sound, graphics and window management and many other things for you on all the platforms you can dream of. This means that this project can be ported to Linux/Mac, etc with ease since I don't use any windows-specific functions. 
You can learn more about it [here](https://www.youtube.com/watch?v=MeMPCSqQ-34).

[GLEW](http://glew.sourceforge.net/) is an extension loading library. 
What this means is that glew checks for the OpenGL capabilities that are present on the machine and sets the appropriate function pointers to them.

[GLM](http://glm.g-truc.net/0.9.5/index.html) is a math library that works with structures similar to the ones found in GLSL.

All the initializations take place in ```InitSystem()```. 
This creates an window, sets a few OpenGL-related attributes and grabs a window context with ```SDL_GL_CreateContext```. 
As you can see the OpenGL version required for our context is 4.3. 
In my opinion using compatibility profile should be left for the experts since there are a lot of things from the older versions that were deprecated and it's unclear (at least to me) how they interact with the newer additions from core. 
For example, does ```glMemoryBarrier``` also work for vertex buffer objects ( the ones used without Vertex Array Objects? ) Documentation seems to just refer VAOs.

Here's the code from ```InitSystem```:
```c++
SDL_Init(SDL_INIT_VIDEO);
g_pWindow=SDL_CreateWindow("GLParticles",SDL_WINDOWPOS_CENTERED,SDL_WINDOWPOS_CENTERED, 1600, 900, SDL_WINDOW_OPENGL|SDL_WINDOW_SHOWN);

//Specify context flags.
SDL_GL_SetAttribute(SDL_GL_CONTEXT_MAJOR_VERSION, 4);
SDL_GL_SetAttribute(SDL_GL_CONTEXT_MINOR_VERSION, 3);
SDL_GL_SetAttribute(SDL_GL_CONTEXT_PROFILE_MASK, SDL_GL_CONTEXT_PROFILE_CORE);
SDL_GL_SetAttribute(SDL_GL_ACCELERATED_VISUAL, 1);
SDL_GL_SetAttribute(SDL_GL_RED_SIZE, 8);
SDL_GL_SetAttribute(SDL_GL_GREEN_SIZE, 8);
SDL_GL_SetAttribute(SDL_GL_BLUE_SIZE, 8);
SDL_GL_SetAttribute(SDL_GL_ALPHA_SIZE, 8);
SDL_GL_SetAttribute(SDL_GL_DOUBLEBUFFER, 1);

#ifdef DEBUG_OPENGL
SDL_GL_SetAttribute(SDL_GL_CONTEXT_FLAGS,SDL_GL_CONTEXT_DEBUG_FLAG);
#endif

//Create opengl context
gContext = SDL_GL_CreateContext(g_pWindow);

glewExperimental = GL_TRUE;
glewInit();
```

Now comes what I consider an important part: debugging OpenGL :). I've worked with both [Nvidia Nsight](https://developer.nvidia.com/nvidia-nsight-visual-studio-edition) and [AMD GPU PerfStudio](http://developer.amd.com/tools-and-sdks/graphics-development/gpu-tools/gpu-perfstudio-2/) and both tools are about the same quality and have almost the same features. Nvidia's NSight has shader debug capabilities (it does not support compute shaders debugging unfortunately) while GPU Perf Studio can't step through your shaders. But the most helpful debug tool are debug messages. They are initialized in the last part of the ```InitSystem``` function like this:
```
#ifdef DEBUG_OPENGL
   glEnable(GL_DEBUG_OUTPUT_SYNCHRONOUS );
   glDebugMessageCallback(openglDebugCallback, NULL);
   glEnable( GL_DEBUG_OUTPUT);
#endif // DEBUG_OPENGL
```

They are useful beyond belief. In openglDebugCallback I print the message and call ``` __debugbreak()```. No errors or warnings allowed policy :). Just for reference, the callback function has the following prototype:

```void APIENTRY openglDebugCallback (GLenum source, GLenum type, GLuint id, GLenum severity, GLsizei length, const GLchar* message, void* userParam)```

Now we have all of our systems initialized.

Moving on, ParticleSystem is the class that does the actual job of drawing particles and updating them. There are three important methods in this class: ```Init```, ```Update``` and ```Render```. Let's go through them in that order.

```ParticleSystem::Init``` is called only once and as the name says it will handle the initialization of our internal structures. Init first initializes two temporary arrays with the starting positions and velocities for the particles. After doing this it calls ```RenderInit``` that handles the initialization for OpenGL-related members. The code from ```RenderInit``` looks like this:
```c++
//Initialize and create the compute shader that will move the particles in the scene
m_ComputeShader.Init();

m_ComputeShader.CompileShaderFromFile("Shaders\\ComputeShader.glsl", GLShaderProgram::Compute);
m_ComputeShader.Link();

m_glPositionBuffer[0]=AllocateBuffer(GL_SHADER_STORAGE_BUFFER,(float*)particlesPos, m_ParticleCount* sizeof(ParticlePos));
m_glPositionBuffer[1]=AllocateBuffer(GL_SHADER_STORAGE_BUFFER,(float*)particlesPos, m_ParticleCount* sizeof(ParticlePos));

m_glVelocityBuffer[0]=AllocateBuffer(GL_SHADER_STORAGE_BUFFER,(float*)particlesVelocity, m_ParticleCount* sizeof(ParticleVelocity));
m_glVelocityBuffer[1]=AllocateBuffer(GL_SHADER_STORAGE_BUFFER,(float*)particlesVelocity, m_ParticleCount* sizeof(ParticleVelocity));

//Cache uniforms
m_glUniformDT = m_ComputeShader.GetUniformLocation("dt");
m_glUniformSpheres = m_ComputeShader.GetUniformLocation("spheres[0].sphereOffset");

//Create and set the vertex array objects
glGenVertexArrays(2, m_glDrawVAO);

glBindVertexArray(m_glDrawVAO[0]);
glBindBuffer(GL_ARRAY_BUFFER, m_glPositionBuffer[0]);
glVertexAttribPointer(0, 4, GL_FLOAT, GL_FALSE, 0, 0);
glEnableVertexAttribArray(0);

glBindVertexArray(m_glDrawVAO[1]);
glBindBuffer(GL_ARRAY_BUFFER, m_glPositionBuffer[1]);
glVertexAttribPointer(0, 4, GL_FLOAT, GL_FALSE, 0, 0);
glEnableVertexAttribArray(0);
```

The first part allocates 4 buffers for velocities and positions. The Shader Storage Buffer Objects are also initialized with data from the two arrays that we just generated in ```ParticleSystem::Init```. In theory only one SSBO for velocity and one for position for is sufficient since you can also write back to them. In practice this crashes on certain drivers for slightly older cards (the ATI 6970 I have at home for example). In order to make this work on a wider range of video cards I've chosen to double buffer them. Next up comes the initialization of 2 vertex array objects, one for each of the position buffers. When drawing, I just switch to the one that contains the output from the compute shader.

Next up comes ```ParticleSystem::Update```. It has the role of computing the new positions and velocities the particles. The important part of the ```Update``` function looks like this:
```
glUseProgram(m_ComputeShader.GetHandle());

glBindBufferRange(GL_SHADER_STORAGE_BUFFER,0,m_glPositionBuffer[!m_csOutputIdx],0,m_ParticleCount* sizeof(ParticlePos));
glBindBufferRange(GL_SHADER_STORAGE_BUFFER,1,m_glVelocityBuffer[!m_csOutputIdx],0,m_ParticleCount* sizeof(ParticleVelocity));

glBindBufferRange(GL_SHADER_STORAGE_BUFFER,2,m_glPositionBuffer[m_csOutputIdx],0,m_ParticleCount* sizeof(ParticlePos));
glBindBufferRange(GL_SHADER_STORAGE_BUFFER,3,m_glVelocityBuffer[m_csOutputIdx],0,m_ParticleCount* sizeof(ParticleVelocity));

glDispatchCompute(m_NumWorkGroups[0], m_NumWorkGroups[1], m_NumWorkGroups[2]);
glMemoryBarrier(GL_VERTEX_ATTRIB_ARRAY_BARRIER_BIT);
```

The magic numbers 0, 1, 2 and 3 come from the layout( binding = … ) declarations that can be found in the compute shader:
```c++
layout ( binding = 0 ) buffer buffer_InPos {
vec4 InPos[];
};
layout ( binding = 1 ) buffer buffer_InVelocity {
vec4 InVelocity[];
};
layout ( binding = 2 ) buffer buffer_OutPos {
vec4 OutPos[];
};
layout ( binding = 3 ) buffer buffer_OutVelocity {
vec4 OutVelocity[];
};
```

The following call to ```glDispatchCompute``` kicks off the the compute shader. The parameters for this function represent the workgoup count on each of the 3 axes. Compute shaders are organized in this way:
- There is a global workgroup that and this global workgroup is composed of local workgroups.
- The number of local workgroups on x,y,z are specified as input to ```glDispatchCompute```.
- In turn, each local workgroup is composed of work items. A work item can be thought as an actual execution of the compute shader.
- The number of work items in a local workgroup is defined in the compute shader like this: layout( local_size_x = 32, local_size_y = 32, local_size_z = 1) in;
- Both local and global workgroups are defined on 3 axes.

In the beginning of this rather long article I've mentioned that compute shaders have no input or output. The call to ```glMemoryBarrier``` at the end of the ```Update``` function is there because I want to read the updated values when drawing the particles. This means that I always read the values after the compute shader is done updating them.

And now for the last part in this article, drawing the particles. The output buffer is used as an input for the geometry pipeline. I use it to draw point sprites. And below we have the ```Update``` function:
```c++
glUseProgram( glDrawShaderID);

//Set the active Vertex array object
glBindVertexArray(m_glDrawVAO [m_csOutputIdx]);

//Draw
glDrawArrays( GL_POINTS, 0, m_ParticleCount );
```

That's it. It just selects the shader program, binds the vertex array object and issues a glDrawArrays call. And this is the end result:

[TODO] - Add image here

Right now particles also collide with some spheres. These spheres are actually just a position and a radius that are sent to the compute shader as of uniforms. At the beginning of the program I just spawn them randomly. The compute shader itself is made by a lot of hacks and hardcoded parts. Each compute shader invocation takes care of updating a single particle. One could imagine having it update let's say 16 particles inside a for loop, and thus lowering the total number of workgroups needed.

Another interesting thing that could be added would be collisions with more complex shapes, actual meshes not just spheres or simple shapes. If I'll get some free time I will also investigate ideas in this direction. First idea that I had was to construct a distance field representation from a mesh and then feed it to the compute shader.

I hope that this provides some insights for people just starting to learn OpenGL that are interested in experimenting with compute shaders or large amounts of particles.
