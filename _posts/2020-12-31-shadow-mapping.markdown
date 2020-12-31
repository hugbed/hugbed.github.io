---
layout: post
title:  "Directional Light Shadow Map Transform"
date:   2020-12-31 3:09:53 -0500
categories: jekyll update
permalink: /directional_light_shadow_map_transform.html
---

See code example on GitHub: [webgl-samples](https://github.com/hugbed/webgl-samples).

### Motivations

There are already many tutorials on the internet and articles showing how to implement shadow mapping. However, most of them do not go into too many details on how to dynamically compute the light transform and instead use a fixed transform that works well for the demo scene. Since the light's transform directly affects shadow quality, a logical next step after implementing shadow mapping would be to understand how to dynamically compute this light transform to maximize shadow quality. Understanding this will also be helpful to  implement more advanced shadow mapping algorithms such as Cascaded Shadow Maps.

### Shadow Mapping Algorithm

First, let's summarize the basic shadow mapping algorithm:

1. Render a depth buffer per light in a separate render pass
     * Compute the view and projection transforms for your light (orthographic projection for directional lights or perspective for point, spotlights)
     * Render your scene in a separate render pass onto a depth buffer for your light using this transform as the view + projection matrix instead of your camera view + projection transforms.
     * Repeat this if your technique requires multiple depth buffers (e.g. point lights use a cube map, cascaded shadow maps use multiple depth maps).
2. Use this/these depth buffer in your material pass to determine if your fragment is in shadow or not
     * Transform your world fragment into light space using your light's view and projection matrices. This can be done either in the vertex or the fragment shader. Note that if you have a lot of lights, it might be easier to do it in the fragment shader, else you'll have to transfer all these light fragments from the vertex to the fragment shader.
     * Use this transformed coordinate now in clip space to sample the light's depth texture.
     * Compare this depth with your current fragment depth to determine if the fragment is in shadow or not. Usually a bias is used here to prevent shadow acne and PCF filtering to improve shadow quality.
     * Render your fragment without the diffuse and specular contributions if it's in shadow (only ambient light remains).

A good tutorial on this subject is [this tutorial from learnopengl.com](https://learnopengl.com/Advanced-Lighting/Shadows/Shadow-Mapping). [This article from Microsoft](https://docs.microsoft.com/en-us/windows/win32/dxtecharts/common-techniques-to-improve-shadow-depth-maps) also contains insightful information on how to improve shadow quality.

This articles focuses on how to compute the orthographic transform for directional lights to maximize shadow quality. This article will not go into too much details on the actual shadow mapping algorithm so please refer to other shadow mapping tutorials as necessary.

### The Light Transform

As mentioned earlier, directional lights use an orthographic projection to render objects to a shadow map. This orthographic projection can be derived directly from an axis aligned bounding box in the light's view coordinate system. This bounding box defines which objects needs to be rendered onto the shadow map. All objects inside this bounding box will be rendered onto one face of this box, usually on the plane facing in *-z* in local light coordinates.

Let's start with a simple example to show how this work.

Let's say you have objects around the world's origin, all within 10 meter around the origin and you want to render shadows for all of them for a directional light looking straight down.

![Scene to render](/assets/images/2020-12-31-shadow-mapping/doodle1.png)

You'll need to define a 10 meter bounding box that you'll then transform into light space to build your view and projection matrices. Let's define the world system coordinate with y axis pointing up and the forward vector of our light pointing at -z in the light's local coordinate system.

![Bounding box to find](/assets/images/2020-12-31-shadow-mapping/doodle2.png)

#### View Transform

First, let's compute the view transform for this light. Since all we need is an axis aligned bounding box, the position part of the view transform can be chosen arbitrarily, as long as your bounding box is defined relative to this position. With this in mind, let's choose a position of (0, 0, 0). Note that this works even if the bounding box is not centered around the origin.

Next, we need a rotation matrix, which can be defined by 3 orthonormal vectors. The first vector is our light direction vector which points down:

```javascript
const lightDir = vec3.fromValues(0.0, -1.0, 0.0);
```

We can then choose the up and right vector arbitrarily. We choose the right vector as *x* if it's not parallel to our light direction, else *y*:

```javascript
let right = vec3.fromValues(1.0, 0.0, 0.0);
if (Math.abs(vec3.dot(lightDir, right)) > 0.9999) {
    right = vec3.fromValues(0.0, 0.0, 1.0);
}
```

Then we choose the up vector to be the cross product of the light direction and the right vector, to have a valid orthonormal rotation matrix:

```javascript
const up = vec3.create();
vec3.cross(up, lightDir, right);
```

We now have the 3 vectors we need to build the view transform:

```javascript
var viewMatrix = mat4.create();
mat4.lookAt(viewMatrix,
    vec3.create(), // eye (position)
    lightDir, // center (direction)
    up
);
```

#### Orthographic Projection

To build the orthographic project matrix, we start with our bounding box in world coordinates:

```javascript
var boxWorld = new BoundingBox();
boxWorld.min = vec3.fromValues(-10, -10, -10);
boxWorld.max = vec3.fromValues(10, 10, 10);
```

Then transform this bounding box in the lights' view coordinates:

```javascript
var boxLight = boxWorld.clone();
boxLight.transform(viewMatrix);
```

Then we can use this directly to compute the projection matrix:

```javascript
var projectionMatrix = mat4.create();
mat4.ortho(projectionMatrix,
    boxLight.min[0], boxLight.max[0],
    boxLight.min[1], boxLight.max[1],
    -boxLight.max[2], -boxLight.min[2] // looking at -z
);
```

Note: if you're in C++ using glm, you don't need to invert the z axis like it's done here, you can use boxLight.min.z, boxLight.max.z directly. I suppose this is because of the handedness of the transforms on gl-matrix vs glm.

So there we have it, then you can pass these transforms to your shaders and transform from world to light clip space with:

```glsl
highp vec4 p_light = projectionMatrix * viewMatrix * vec4(p_world, 1.0); 
```

### Computing the bounding box

The axis-aligned bounding box can be defined as a min and a max value. To compute this bounding box for an object, you just have to take the min/max vertex values for your mesh. You can then transform this bounding box into world coordinates.

### Applying a coordinate transform to a bounding box

To transform a bounding box from one coordinate system (e.g. world) to another (e.g. light):

1. Compute the 8 corners of your bounding box

```js
getCorners() {
    return [
        vec3.fromValues(this.min[0], this.min[1], this.min[2]),
        vec3.fromValues(this.min[0], this.min[1], this.max[2]),
        vec3.fromValues(this.min[0], this.max[1], this.min[2]),
        vec3.fromValues(this.min[0], this.max[1], this.max[2]),
        vec3.fromValues(this.max[0], this.min[1], this.min[2]),
        vec3.fromValues(this.max[0], this.min[1], this.max[2]),
        vec3.fromValues(this.max[0], this.max[1], this.min[2]),
        vec3.fromValues(this.max[0], this.max[1], this.max[2])
    ];
}
```

2. Transform each of these corners into the new coordinate system using the transform matrix.

```js
transform(transformMatrix) {
    // We can't just take the axis-aligned min/max we have
    // since this only applies to this coordinate
    // system. We need to reproject all box corners.
    const corners = this.getCorners();

    this.reset();
    for (const corner3 of corners) {
        // p = T * corner;
        const p4 = vec4.create();
        const corner4 = vec4.fromValues(
            corner3[0], corner3[1], corner3[2], 1.0);
        vec4.transformMat4(p4, corner4, transformMatrix);

        // ...
    }
}
```

3. Take the min/max of the transformed points to get the new bounding box.


```js
transform(transformMatrix) {
    // We can't just take the min/max we have
    // since this only applies to this coordinate
    // system. We need to reproject all box corners.
    const corners = this.getCorners();

    this.reset();
    for (const corner3 of corners) {
        // p = T * corner;
        // ...

        // keep min/max
        const p3 = vec3.fromValues(
            p4[0]/p4[3],
            p4[1]/p4[3],
            p4[2]/p4[3]);
        vec3.min(this.min, this.min, p3);
        vec3.max(this.max, this.max, p3);
    }
}
```

See [bounding_box.js](https://github.com/hugbed/webgl-samples/blob/master/src/bounding_box.js) for the implementation.

### Choosing a bounding box

This technique can be used to define any bounding box for which you want to render shadows. For each object in your scene, you can compute its bounding box and combine the boxes of all the objects you need shadows for as one big bounding box. Then this box is transformed into the light's view space to compute the light's transform.

Now which objects should we include in this bounding box? We have shadow casters (cubes in this example) and shadow receivers (the plane in this example).

#### All objects in the scene

We can start by including all shadow casters and shadow receivers to make sure that all shadows are rendered on screen. You'll get something like this:

![Large plane in bounding box](/assets/images/2020-12-31-shadow-mapping/large_plane.png)

We see that including shadow receivers is useless since we don't need to render them to the depth map. We then have less resolution for the objects that we actually need and this affects the shadow quality. If the scene is very large, including all objects in the bounding box can give very pixelated shadows.

![Very large plane in bounding box](/assets/images/2020-12-31-shadow-mapping/very_large_plane.png)

Each cube there uses only a couple of fragments in the shadow map so when we reproject these fragments on the camera, a lot of fragments share the same depth, hence these pixelated shadows.

#### Shadow casters only

So what if we render all shadow casters to the depth map and ignore shadow receivers?

![Shadow casters only](/assets/images/2020-12-31-shadow-mapping/shadow_casters_only.png)

That's pretty good! There is no more wasted space around objects we don't need to render to the depth map. The bounding box used is shown in green here.

However, if there are objects outside the view frustrum, depth map fragments will be wasted rendering these objects to the depth map since we won't see those shadows:

![Shadow caster outside](/assets/images/2020-12-31-shadow-mapping/shadow_caster_outside.png)

#### Camera frustrum only

To improve on this solution, one could render to the shadow map, only objects inside the view frustrum. We'll assume for now that objects outside the view frustrum do not cast shadow onto it. First, the camera frustrum bounding box needs to be determined. Four camera points for the near plane and four for the far planes can be reprojected in world coordinates, then min/max'ed to get a bounding box (see [ShadowMap.updateTransforms](https://github.com/hugbed/webgl-samples/blob/master/src/shadow_map.js) and [Camera.computeFrustrumCorners](https://github.com/hugbed/webgl-samples/blob/master/src/camera.js)):

```javascript
let camCorners = camera.computeFrustrumCorners();
let camBox = BoundingBox.fromPoints(camCorners);
```

Then, for each shadow caster, if their bounding box intersects the camera frustrum bounding box, you include it into the shadow caster bounding box (see [bounding_box.js](https://github.com/hugbed/webgl-samples/blob/master/src/bounding_box.js) for bounding box intersection):

```javascript
for (const obj of objects) {
    const objBox = obj.getWorldBoundingBox();
    if (objBox.intersects(camBox)) {
        vec3.min(box.min, box.min, objBox.min);
        vec3.max(box.max, box.max, objBox.max);
    }
}
```

We see that it solves the issue if the cube is far on the right, as the cube is not included in the shadow map:

![Bounding box using only objects in the camera frustrum](/assets/images/2020-12-31-shadow-mapping/camera-frustrum-only.png)

However if the cube was high up instead of far on the right, we would be missing its shadow. Our assumption that objects
outside the camera frustrum do not cast shadow onto the camera frustrum was wrong. Objects out of the camera frustrum,
but between the light and the camera frustrum must also be included so that they properly cast shadows. 

#### Combined camera frustrum and shadow casters

To include these objects, we need to extend their bounding box in the direction of the light, up to the boundary of the scene.

Here's the final object culling and bounding box computation for shadow casters.

```javascript
    // Compute the whole scene bounding box
    let sceneBox = new BoundingBox();
    for (const obj of objects) {
        const objBox = obj.getWorldBoundingBox();
        vec3.min(sceneBox.min, sceneBox.min, objBox.min);
        vec3.max(sceneBox.max, sceneBox.max, objBox.max);
    }

    let camCorners = camera.computeFrustrumCorners();
    let camBox = BoundingBox.fromPoints(camCorners);
    
    const shadowViewMatrix = shadowMap.getViewMatrix();

    // Transform sceneBox in shadow view space
    sceneBox.transform(shadowViewMatrix);

    // Transform camBox in shadow view space
    camBox.transform(shadowViewMatrix);

    // Stretch the camera bounding box towards the light
    // (opposite of light direction) so that it contains the whole scene.
    // Note: light direction is -z in local space
    camBox.max[2] = sceneBox.max[2] 

    // Transform this box back to world coordinates
    let shadowViewInverse = mat4.create();
    mat4.invert(shadowViewInverse, shadowViewMatrix);
    sceneBox.transform(shadowViewInverse);

    // Then compute shadow caster box using
    // this streched camera frustrum bounding box
    box.reset();
    for (const obj of objects) {
        const objBox = obj.getWorldBoundingBox();
        // Only include 
        if (objBox.intersects(camBox)) {
            vec3.min(box.min, box.min, objBox.min);
            vec3.max(box.max, box.max, objBox.max);
        }
    }
```

And here it is, the cube up high now does cast shadow onto the view frustrum!

![Bounding box with extended view frustrum in the direction of the light](/assets/images/2020-12-31-shadow-mapping/camera-frustrum-extended-up.png)

But is still not included in the shadow map when it doesn't cast shadow (when far on the right):

![Bounding box with extended view frustrum in the direction of the light](/assets/images/2020-12-31-shadow-mapping/camera-frustrum-extended-right.png)

## Conclusions

So that's it! Next step could be to implement cascaded shadow maps, which uses multiple shadow maps to have higher resolution of the depth texture near the viewer and lower resolution far away.