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

Transforming a texture uv coordinate into a world coordinate looks like this:

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

We can then use the function like so:

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

Now I'm fairly certain that this is as far as it goes in TotK and they're just smart with how they mask it with textures, but the issues with this technique start to pop up when you `step` the sine function.

![]()

This sudden speed-up is happening because we're essentially colorizing the outline of an expanding sphere, and when it hits straight edges like that it rapidly takes up a lot of the space. But this doesn't look right when we consider that the particles are meant to be expanding from the players feet outwards, so they shouldn't suddenly gain lots of velocity when moving around an edge. In TotK, they actually *do* do this when they hit walls at sharp 90º angles, it's just hard to tell because they're already moving at a relatively slow speed anyways, and clever masking with textures.  

The solution to this is to somehow compute the actual real "travelling" distance they'd need to travel, instead of an approximation to it by just taking the direct euclidean distance. One issue. Practically impossible. But here was my idea:

1. Lerp a vector between `u_position` and `worldPosition` in a numeric integral of `t` between 0 and 1
2. Take said vector, reproject it, then use that projected vector's `.xy` coordinates to sample the depth buffer
3. Form a new vector by passing the depth value sampled into `depthToWorld`
4. Take the distance between the new vector and the previous vector (initially `u_position`)
5. Add that distance onto the final distance sum
6. Repeat for all values of `t` between 0 and 1 using an appropriate `dt`

Essentially, when we just take the euclidean distance, we're making the (implicit) assumption that the difference between the lerp'd vector between `u_position` and `worldPosition` and the actual vector that is on the geometry is negligible enough to not be accounted for.

```glsl
float travellingDistanceBetween(vec3 a, vec3 b)
{
    const float dt = 0.05;
    
    float result = 0.0;
    
    vec3 prevVector = a;
    for (float t = 0.0; t <= 1.0; t += dt) {
        vec3 c = a + t*(b-a);
        vec4 reprojected = u_projMatrix * u_viewMatrix * vec4(c, 1.0);
        reprojected.xyz /= reprojected.w;
        vec2 screenCoordinate = (reprojected.xy + 1.0) * 0.5;
        vec3 timeStepWorldCoordinate = depthToWorld(screenCoordinate);
        result += length(timeStepWorldCoordinate - prevVector);
        prevVector = timeStepWorldCoordinate;
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

![]()

This way you get somewhat of an approximation of the travelling distance. The big issue is that it's completely relative to the camera, and on top of that it only works if all pixels that it can travel between are visible, so the use cases for this would be minimal if any exist at all.

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