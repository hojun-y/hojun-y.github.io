---
layout: post
title:  "Visualizing Slope Field using Houdini"
tags: ["visualization", "houdini"]
comments: True
---
<img src="{{site.baseurl}}/assets/visualizing-slope-field/image.png" alt="Rendered image" id="rendered-image"/>
In this project, I visualized the slope field of the famous [Lotka-Volterra equations](https://en.wikipedia.org/wiki/Lotka%E2%80%93Volterra_equations). It represents the change in population between predators and preys, where predators' food supply is totally dependent on preys. The relationship between them is given as:

$$
\left\{
  \begin{array}{ll}
    \frac{dx}{dt}=\alpha x-\beta xy\\
    \frac{dy}{dt}=\gamma xy-\delta y\\
  \end{array}
\right.
$$

$x$ and $y$ are the populations of preys and predators, respectively. Other coefficients determine the interaction between two species as noted below.

| Coefficient | Meaning |
|---|---|
|$\alpha$|Growth rate of prey|
|$\beta$|Predation rate of prey|
|$\gamma$|Growth rate of predator|
|$\delta$|Death rate of predator|

As time evolves, the prey's population increases($\alpha x$), but they are eaten by predators($-\beta xy$) at the same time. Also, predator's population increases($\gamma xy$) proportional to the abundance of food(i.e. number of prey) while some of them will die($-\delta y$).

### Visualization Workflow
#### Creating instancing points
To place each signpost in the right direction, I needed to calculate the direction vector for each instancing point.$<\frac{dx}{dt}, 0, \frac{dy}{dt}>$. I used `attribute wrangle` to assign a normalized direction vector to each point. Since the points are lying on the XZ-plane, I used Houdini's z coordinate(`P.z`) as a $y$ value in the  equation.
```c++
// VEXpression code in wrangle
float alpha = chf("alpha");
float beta = chf("beta");
float gamma = chf("gamma");
float delta = chf("delta");
float scale = chf("scale");

vector p = v@P * scale;
float dx = alpha * p.x - beta * p.x * p.z;
float dy = delta * p.x * p.z - gamma * p.z;
v@dir = set(dx, 0, dy);
v@dirNormalized = normalize(v@dir);
```

<figure style="width:80%">
<img src="{{site.baseurl}}/assets/visualizing-slope-field/field.PNG" alt="Slope field"/>
<figcaption>Slope field(dirNormalized attribute)</figcaption>
</figure>

To add more dynamic feeling to the image, I elevated signposts according to the log-scaled magnitude of its direction vector. Big base value($5\times 10^3)$ was needed because the direction vector has quite a big value.
#### Instancing objects
Cylinders are procedurally modeled in Houdini. I cut the top part of the cylinder by subtracting slanted box. The signpost is just simply modeled in Blender.

<figure style="width:80%">
<img src="{{site.baseurl}}/assets/visualizing-slope-field/cylinder.png" alt="Node setup for the cylinder"/>
<figcaption>Node setup for the cylinder</figcaption>
</figure>

#### Rendering
I used Octane renderer with DL kernel option. I like Octane because of its speed and low subscription cost. (Although there are still some bugs)
After rendering, I added more indirect reflection to the beauty pass. I also added bokeh effects using gaussian-blurred Z map.

<figure style="width:80%">
<img src="{{site.baseurl}}/assets/visualizing-slope-field/lighting.PNG" alt="Lighting setup"/>
<figcaption>Lighting setup</figcaption>
</figure>

### Future outlook & What I Learned
This is my first project using Houdini as a primary tool. I learned how to instance objects, how to set up materials and lights. I am so glad that I finished all of the processes successfully despite of a lot of obstacles I encountered(e.g. Wrong Z-depth map, unsatisfying lighting setup, etc.) In the next project, I will focus on more mathematical fundamentals under the hood.

### References
1. [https://mathworld.wolfram.com/Lotka-VolterraEquations.html](https://mathworld.wolfram.com/Lotka-VolterraEquations.html)
2. [https://en.wikipedia.org/wiki/Lotka-Volterra_equations](https://en.wikipedia.org/wiki/Lotka%E2%80%93Volterra_equations)
