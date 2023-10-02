# Fixed Uniform Load Beam Derivation

So I've recently been reading up on shear-moment diagrams and cantilever beams due to it being somewhat related to my EPQ for school (to anyone not British, it's basically a thing some schools force students to do when they build an artefact (make something) or do a dissertation (write something) on whatever you want).

Anyway I was quite dissapointed to find that pretty much all derivations I could find of the special case of a uniformally distributed load along a beam fixed at both ends were fairly... bad, and didn't really care to explain where they were coming from. Also doesn't help all the videos featuring it were filmed in 120p with about 0.2 frames per hour.

After spending literally an embarrassingly large chunk of my day banging my head against a blank sheet of paper, here's my derivation (by double integration method):

## The Derivation

Since we have an equal resultant force on both ends of the fixed beam, call it $R$, and the uniformally distributed load $q$ is along the beam of length $L$, it means that $R + R = qL$, so therefore $R = \frac{qL}{2}$.

This means that our shear function can be written as so: $V(x) = \frac{ql}{2} - qx$.

Since $M(x) = \int{V(x) \ \mathrm{d} x}$, integrating we get $M(x) = -\frac{q}{2} x \left( x - L \right) + C_1$

By DI method, $\frac{\mathrm{d}^2 y}{\mathrm{d} x^2}=\frac{M}{EI}$, so:

$y' = \frac{1}{EI} \int{M(x) \ \mathrm{d} x}$
$= \frac{1}{EI} \left( -\frac{q}{6} x^3 + \frac{qL}{4} x^2 + C_1 x \right) + C_2$ but $C_2 = 0$ by boundary condition $y'(0) = 0$
$y = \frac{1}{EI} \left( -\frac{q}{24} x^4 + \frac{qL}{12} x^3 + \frac{C_1}{2} x^2 \right) + C_2$ but $C_3 = 0$ by boundary condition $y(0) = 0$

Now we can apply the final boundary condition $y(L) = 0$ and solve for $C_1$ where we find that:

$C_1 = -\frac{qL^2}{12}$

So we know that $M(x) = -\frac{q}{2} \left( x \left( x - L \right) + \frac{L^2}{6} \right)$!

This also has the benefit that now you know $y(x)$, so you can also calculate the deflection at each point in the beam, or just make a cool Desmos graph I guess? ;)

Now you can proceed as normal, most likely calculating the maximum moment applied to the beam.

Thanks for reading!

- Krýša
