---
layout: post
title:  "Procedurally Modeling Embroidery with Houdini"
tags: ["modeling", "houdini"]
comments: True
---
<img src="{{site.baseurl}}/assets/embroidery/image.png" alt="Rendered image" id="rendered-image"/>

### 1. Introduction
Unlike texture-based embroidery patterns, curve-based stitches have several advantages:

* **Proceduralism**
  * Using VEXpression, you can access every stitch’s attributes. This makes you animate(or even simulate) stitches at any frame.
* **Ease of modification**
  * Since all the stitches are placed procedurally, changing the appearance of the pattern is relatively easier than modifying the clothing’s texture map.

### 2. Workflow
As shown below, the entire process has six steps.
1. Determining stitches' Position
2. Intersection test with pattern primitives
3. Intersection test with cloth model
4. Modeling the Details
5. Storing stitches' initial state
6. Deform stitches

### 3. Determining stitches’ Position
To make the calculation simple, the stitches’ direction should be aligned with the x-axis. Thus, pattern primitive should be rotated by the stitching angle(Namely, `angle` primitive attribute). This rotational transform is reversed at the end of the generation process.
```c++
// Prim-level wrangle
float rot = radians(f@angle);
3@xform = ident();
3@xform_inv = ident();
rotate(3@xform, rot, {0, 1, 0});
rotate(3@xform_inv, -rot, {0, 1, 0});

// Point-level wrangle
matrix3 trans = prim(0, "xform", 0);
@P *= trans;
```

Now we can calculate how many stitches are needed to fill the entire pattern. Since the minimal spacing between two threads is $2r$ (where r is the radius of the thread), $\lfloor \frac{l}{2r}\rfloor$ stitches can be placed without overlapping.

<figure style="width:50%">
<img src="{{site.baseurl}}/assets/embroidery/stitch_positioning.png"/>
<figcaption>Fig.2 : Calculating points' position</figcaption>
</figure>

```c++
v@size = getbbox_size(0);  // Size of pattern primitive
i@n = floor(v@size.z / (2 * f@r)) + 1;
```

I used a grid node to generate stitches. To match the spacing between the grid’s primitives, I set the ‘rows’ parameter to $\lfloor \frac{l}{2r}\rfloor+1$ and deleted the last primitive.

### 4. Handling Intersections
For each stitch, I calculated the intersection points between stitches and the embroidery pattern. These points determine the final shape of the stitch. After this step, a small jitter is applied to the stitches’ position. (This makes the embroideries’ appearance more natural)

Some stitches may be placed over multiple primitives of cloth. Each segment of the curve should not overlap two or more primitives of the underlying cloth. So I divided each curve at the intersection point with the clothes. This requires a lot of computation time, but it is necessary when animating the stitches.

### 5. Modeling the Details
Real-life embroidery stitches have a “round” endpoint. To achieve this appearance, I added two points at each endpoint of the curve. Then, I moved the curve's points according to Fig.3.

<figure style="width:80%">
<img src="{{site.baseurl}}/assets/embroidery/stitch_trig.png"/>
<figcaption>Fig.3 : Calculating points' position</figcaption>
</figure>

```c++
// Input 0 : Stitch prim
// Input 1 : Pattern prim
float theta = prim(1, "theta", 0);
float r = prim(1, "r", 0);

vector start = point(0, "P", 0);
vector end = point(0, "P", @numpt-1);
float rsint = sin(theta);
float rcost = r * cos(theta);
float rcsct = r * (1 / rsint);
rsint *= r;

vector2 endOffset = set((2 * rcsct - rsint), -rcost);
vector2 roundingOffset = set(rsint, rcost);

setpointattrib(0, "P", 0, set(start.x - endOffset.x, endOffset.y, start.z));
setpointattrib(0, "P", 1, set(start.x - roundingOffset.x, roundingOffset.y, start.z));
setpointattrib(0, "P", @numpt-2, set(end.x + roundingOffset.x, roundingOffset.y, end.z));
setpointattrib(0, "P", @numpt-1, set(end.x + endOffset.x, endOffset.y, end.z));

for (int i = 2; i < @numpt-2; i++){
    vector p = point(0, "P", i);
    setpointattrib(0, "P", i, set(p.x, r, p.z));
}
```

<figure style="width:80%">
<img src="{{site.baseurl}}/assets/embroidery/detail_modeling.png"/>
<figcaption>Fig.4 : Final shape of the stitch</figcaption>
</figure>

If the length of the stitch is smaller than $2r\csc \theta$, two endpoints will overlap each other. In that case, the stitch is removed.

### 6. Getting Ready For The Animation
Before deforming the stitches, every point in the stitch has to store the initial position relative to the clothes. (Thus, we can keep track of the underlying clothes’ deformation state)
```c++
// Input 1 : Cloth mesh
vector origin = v@P;
origin.y += 1;
vector p, uvw;
int prim = intersect(1, origin, set(0, -100, 0), p, uvw);
i@prim = prim;
uvw.z = v@P.y;
v@uvw = uvw;
```
In the code, I replaced the W component of the UVW position with the point’s initial elevation. (Since the W component of polygons’ UVW is always 0, we can store more “meaningful” value to it.)


### 7. Deforming Stitches
Deforming(sticking stitches to the clothes) the curves is quite straightforward:
```c++
vector n = primuv(1, "N", i@prim, v@uvw);
@P = primuv(1, "P", i@prim, v@uvw);
@P += n * v@uvw.z;
```
Points are translated to the new position, then shifted along the normal vector of the cloth primitive.

### 8. Conclusion
Due to the intersection testing method, this method cannot handle concave/non-triangular embroidery patterns well. Dealing with these kinds of polygons is slightly trickier. A special algorithm such as the [even-odd rule](https://en.wikipedia.org/wiki/Even%E2%80%93odd_rule) is needed to determine whether the segment of a stitch is inside in the pattern primitive. I did not use this algorithm as I wanted to keep the problem simple.

### 9. Related Links
1. [Making Beautiful Embroidery for "Frozen 2" (SIGGRAPH ‘20 Talks)](https://media.disneyanimation.com/technology/publications/2020/EmbroiderySiggraph20Official.pdf)
