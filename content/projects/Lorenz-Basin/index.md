---
title: "Lorenz Basin of Attraction"
date: 2026-02-19
tags: ["Modeling", "Numerical Methods"]
author: ["Horacio Moreno Montanes"]
description: "An interactive GPU simulation of the Lorenz system's basin of attraction, rendered in full resolution using WebGL2."
summary: "An interactive GPU simulation of the Lorenz system's basin of attraction on the invariant surface y = −x, rendered in full resolution using WebGL2."
cover: 
    image: "cover.png"
    alt: "Lorenz Basin"
    relative: false
---

---

##### Simulation

Each pixel represents a point on the surface $y = -x$ with coordinates $(x, z)$. The color indicates which fixed point — $C^+$ or $C^-$ — the trajectory converges to under the Lorenz system:

$$\dot{x} = \sigma(y - x), \quad \dot{y} = \rho x - y - xz, \quad \dot{z} = xy - \beta z$$

Integration is performed entirely on the GPU using WebGL2 fragment shaders. Euler, RK2, and RK4 methods are available. Parameters $\sigma$, $\rho$, and $\beta$ can be adjusted in real time.

<a href="/lorenz_basin.html" target="_blank" style="display:inline-block; margin: 1em 0; padding: 10px 20px; background:#1a1a2e; border:1px solid #697aff; color:#697aff; font-family:'Courier New',monospace; font-size:13px; text-decoration:none; border-radius:3px;">Open Simulation →</a>
