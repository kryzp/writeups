# Zelda Tears of the Kingdom Ability Effect Shader

## Literature
 * [Decal deferred rendering](https://mtnphil.wordpress.com/2014/05/24/decals-deferred-rendering/)
 * [LearnOpenGL deferred rendering summary](https://learnopengl.com/Advanced-Lighting/Deferred-Shading)
 * [Computing normals without normals - unrelated but pretty cool](https://c0de517e.blogspot.com/2008/10/normals-without-normals.html)

## Intro
One thing that caught my eye when first playing Zelda: Tears of the Kingdom is the shader developed which gets overlayed on the screen when rendering the world when the player uses hand-based abilities:  

 * **Ultrahand**: *The* defining mechanic of TotK, the ability which lets you build machines and other contraptions out of old Zonai equipment.
 * **Fuse**: Lets the player connect a weapon to a world resource to give it new effects.
 * **Ascend**: Where the player can essentially teleport through the ceiling, provided it's close enough.
 * **Autobuild**: Lets the player re-build creations they've saved by using blueprints.

What I found most interesting was the particles which eminate from the players feet, up and *around* the surroundings of the player. It's one of those things that might not be noticable at first, but if they had just made some simple generic 2D sprite effect for that it would have so much of a negative impact on the feeling of using hand abilities. Also it would look garbage, the more you know huh?  

Anyway here's my crack at the effect. I did many different experiments with different techniques to see what fit, some of which I was certain *wasn't* used by TotK but still worth looking into, I think.  

For reference, this is what the rendering code basically looks like:

``` c++
void DeferredRenderer::init()
{
	// create g-buffer textures & depth
	// create meshes
	// load information
}

void DeferredRenderer::render()
{
    // bind our g-buffer
	m_gBuffer.bind();
	
    // clear the g-buffer
    clear(m_clearColour.r, m_clearColour.g, m_clearColour.b);
	
    // draw all of the things in the world that should be affected by the effect
    renderElements();
    
    // draw our effect
    renderEffect();
	
    // stop drawing to our g-buffer
    m_gBuffer.unbind();
    
    // clear the actual default screen
    clear();
    
    // bind our composite shader which takes in all of the seperate attachments as samplers and composites them together to produce the final image
    bindCompositeShader();
    
    // draw quad
    m_quadMesh.render();
    
    // send the depth information from the read fbo to the write fbo
    glBlitFramebuffer(0, 0, WIDTH, HEIGHT, 0, 0, WIDTH, HEIGHT, GL_DEPTH_BUFFER_BIT, GL_NEAREST);
}
```

## Initial Ideas
First important note: Tears of the Kingdom, uses a *predominantly deferred* rendering pipeline. The important thing here is that decals and clip-space-to-world-space shaders are *much* easier when using a deferred rendering pipeline as you can draw it all after you've finished rendering the depth buffer, all in a single call once all the scene geometry is finished. We can just sample the depth buffer at any point and instantly get the world space position of that point, given we know the projection and view matrices in our effect shader and just composite that over our g-buffer.  

Transforming a texture UV coordinate into a world coordinate looks like this:

```glsl
layout (binding = 0) uniform sampler2D u_depthTexture;

layout (location = 0) in vec2 i_texCoord;

uniform mat4 u_viewMatrix;
uniform mat4 u_projMatrix;

vec3 depthToWorld(vec2 uv)
{
	float depthSample = texture(u_depthTexture, uv).x;
	
	vec3 clipSpaceXYZ = vec3(uv, depthSample);
	clipSpaceXYZ = (clipSpaceXYZ * 2.0) - 1.0; // transform from [0, 1] range to [-1, 1] range
	
	vec4 clipSpacePosition = vec4(clipSpaceXYZ, 1.0);
	
	vec4 viewSpacePosition  = inverse(u_projMatrix) * clipSpacePosition;
	vec4 worldSpacePosition = inverse(u_viewMatrix) * viewSpacePosition;
	
	return worldSpacePosition.xyz / worldSpacePosition.w;
}
```

Initially, we might think to just base it on the distance from the particle emit position (`u_position`) to the world position of the pixel:

```glsl
uniform vec3 u_position; // location from where the particles are being emitted
uniform vec3 u_time;

// functions...

void main()
{
    vec3 worldPosition = depthToWorld(i_texCoord);

    float dist = length(worldPosition - u_position);
    
    if (dist > 10.0) {
        discard; // discard any outside of our max distance to stop weird graphics outside of model
    }
    
    float alpha = sin(6.28*dist - u_time);
    alpha = (alpha + 1.0) * 0.5;

    o_fragColour = vec4(1.0, 1.0, 1.0, alpha);
}
```

Which yields this effect:

![](https://github.com/kryzp/writeups/blob/main/res/totk_ability_effect_shader_0.png)

It looks alright, but there's definately something off about it. The issues with this technique start to pop up when you `step` the sine function.

https://github.com/kryzp/writeups/assets/48471657/a6128df6-1c06-4093-bcd0-9df61842a0f5

This sudden speed-up is happening because we're essentially colorizing the outline of an expanding sphere, and when it hits straight edges like that it rapidly takes up a lot of the space. But this doesn't look right when we consider that the particles are meant to be expanding from the players feet outwards, so they shouldn't suddenly gain lots of velocity when moving around an edge. In TotK, they actually *do* do this when they hit walls at sharp 90º angles, it's just hard to tell because they're already moving at a relatively slow speed anyways, and clever masking with textures.  

The solution to this is to somehow compute the actual real "travelling" distance they'd need to travel, instead of an approximation to it by just taking the direct euclidean distance. If you want maths terms, we're looking for a function $s(\vec{a}, \vec{b}) \to \mathbb{R}$ which gives us the real minimum distance between position vectors $\vec{a}$ and $\vec{b}$.  

My first idea:
1. Lerp a vector between `u_position` and `worldPosition` in a numeric integral of `t` between 0 and 1
2. Take said vector, reproject it, then use that projected vector's `.xy` coordinates to sample the depth buffer
3. Form a new vector by passing the depth value sampled into `depthToWorld`
4. Take the distance between the new vector and the previous vector (initially `u_position`)
5. Add that distance onto the final distance sum
6. Repeat for all values of `t` between 0 and 1 using an appropriate `dt`

```glsl
float travellingDistanceBetween(vec3 a, vec3 b)
{
    const float dt = 0.05;
    float result = 0.0;
    vec3 currVec = a;

    for (float t = 0.0 ; t <= 1.0; t += dt)
    {
        vec3 c = a + t*(b-a);
        vec4 reproj = u_projMatrix * u_viewMatrix * vec4(c, 1.0);
        reproj /= reproj.w;
        vec2 screenCoord = (reproj.xy + 1.0) * 0.5;
        vec4 d = depthToWorld(screenCoord);
        vec4 diff = d - currVec;
        result += length(diff);
        currVec = d;
    }

    return result;
}

// ...

void main()
{
    vec3 worldPosition = depthToWorld(i_texCoord);

    float dist = travellingDistanceBetween(worldPosition, u_position);
    
    if (dist > 10.0) {
        discard; // discard any outside of our max distance to stop weird graphics outside of model
    }
    
    float alpha = sin(6.28*dist - u_time);
    alpha = (alpha + 1.0) * 0.5;

    o_fragColour = vec4(1.0, 1.0, 1.0, alpha);
}
```

It was worth trying but it basically sucks. The big issue is that it's completely relative to the camera, and on top of that it only works if all pixels that it can travel between are visible, so the use cases for this would be minimal if any exist at all.  

I also messed around with using the manhattan distance. This makes the assumption our entire world is blocky and *only* uses straight edges. It also isn't a circle, but hey! It nicely wraps around all of the world geometry without changing speed.

```glsl
float manhattanDistanceBetween(vec3 a, vec3 b)
{
	vec3 d = abs(a - b);
	return d.x + d.y + d.z;
}

// ...

float dist = manhattanDistanceBetween(worldPosition, u_position);
```

![](https://github.com/kryzp/writeups/blob/main/res/totk_ability_effect_shader_1.png)

## Got it working...

Finally, I actually got out some paper and figured out something (*I'm really proud of*), which yields:

![](https://github.com/kryzp/writeups/blob/main/res/totk_ability_effect_shader_4.png)

This works pretty damn well. The particles naturally curve around the terrain without sudden jumps in speed and without any jittery behaviour. It even passes the `step` function test:

![](https://github.com/kryzp/writeups/blob/main/res/totk_ability_effect_shader_3.png)

The code:

```glsl
layout (binding = 1) uniform sampler2D u_normalTexture;

uniform vec3 u_cameraRight;
uniform vec3 u_cameraUp; // usually just (0, 1, 0)

// ...

vec3 worldPosition = depthToWorld(i_texCoord);
vec3 worldNormHere = texture(u_normalTexture, i_texCoord).xyz;

vec3 originToWorldPosition = worldPosition - u_position;

float horiNorm = abs(dot(worldNormHere, u_cameraRight));
float vertNorm = abs(dot(worldNormHere, u_cameraUp));

float lenSqr = horiNorm*horiNorm + vertNorm*vertNorm;
float virtualDistanceSqr = (2.0 * lenSqr) - 1.0;

vec3 vecA = vec3(vec2(horiNorm, vertNorm) * sqrt(max(0.0, -virtualDistanceSqr)), virtualDistanceSqr);
vecA = normalize(vecA);

float distX = originToWorldPosition.x - (vecA.x * originToWorldPosition.y);
float distZ = originToWorldPosition.z - (vecA.z * originToWorldPosition.y);

float dist = sqrt(distX*distX + distZ*distZ);

// ...
```

So, how does it work? Well I essentially get the absolute value of the normal at each pixel. Intuitively, you could imagine two global lights at each side of the normal (left/right or top/bottom) shining down on it, and the value is how bright it is. Basically, imagine a sun light on each side of the viewport and write the horizontal shadows to the red channel, and the vertical shadows to the green channel.

Then I take the sum of their squares which gives me a squared length of the 2D vector described by them. I then transform it into the range $\left[-1, 1\right]$ since it's in $\left[0, 1\right]$ range because that's what the horiNorm and vertNorm encode. We then create a new vector with the $x$, $y$ components being the hori and vert components of the normal mulitplied by the negative virtual distance (clamped to never be below 0) (I'm not really sure what to call `virtualDistance`). The $z$ component is just the virtual distance squared. We then normalize our vector and multiply the y component of our originToWorldPosition vector, then subtract that from the distances along $x$ and $z$, which we then use to calculate the final distance by pythagorean theorem.

## Textures?
Alright, cool, so there's the different ideas/techniques for the distance function, but how do you know what uv coordinates to use?  

Well, my idea was that you sample the effect texture horizontally based on the distance from `u_position` and vertically based on the angle to `u_position`, using the normal at that position as "up" and rotating the world accordingly such that you always have the right angle relative to `u_position` even if you're upside down.  

```glsl
layout (binding = 2) uniform sampler2D u_effectTexture;

// ...

float alpha = exp(-dist); // fade out over time

float texUVx = fract(2.0*dist - u_time); // "wrap" it via fract()

float texUVy = atan(-originToWorldPosition.z, originToWorldPosition.x);
texUVy = clamp(0.5 * (1.0 + (texUVy / 3.14159)), 0.0, 1.0) // change range from [-pi, pi] to [0, 1] uv coordinates

float texCol = texture(u_effectTexture, vec2(texUVx, texUVy)).r * 2.0;
o_fragColour = vec4(vec3(texCol), texCol * alpha);
```

Here's the particle texture:

![](https://github.com/kryzp/writeups/blob/main/res/totk_ability_effect_shader_5.png)

And here's the final result:

![]()

There's obviously a lot more you could do here, such as all the fancy noise and extra effects that get layered on top, the tinting of the world, etc... but I was mostly just interested in this part and I want to move onto other projects, so I'm going to leave it here.  

**Note:** this approach has weird reprecussions `u_position` isn't directly (*or close enough to be negligible*) on the geometry. The "projected" expanding waves will still move but only along the geometry, while `u_position` goes straight through it, which leads to the actual particle emit position being nowhere near `u_position`.

Thank you for reading!  

- Kryštof
