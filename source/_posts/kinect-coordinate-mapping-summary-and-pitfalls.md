---
title: 'Kinect Coordinate Mapping: Summary and Pitfalls'
tags:
  - Computer Graphics
  - Computer Vision
  - Kinect
  - Programming
id: 65
categories:
  - Tech Notes
date: 2017-03-18 23:08:37
header-img: "http://on02twadg.bkt.clouddn.com/kinect.jpg"
catalog: true
mathjax: true
---

I'm using [Microsoft Kinect](https://www.microsoft.com/en-us/kinectforwindows/) for some face tracking applications, and coordinate mapping is often inevitable. However, I found that [the official document](https://msdn.microsoft.com/en-us/library/dn799271.aspx) is quite simple and confusing. Here I will summarize what I figured out while trying to use it. Hopefully it makes things more clear.

<!-- more -->

## Overview

Note: This post addresses [Kinect for Windows v2](https://www.microsoft.com/en-us/kinectforwindows/) and its c++ API in [SDK 2.0](http://www.microsoft.com/en-us/download/details.aspx?id=44561). The conclusions may not apply to previous generation of Kinect or c# API or later version of SDK.

There are three coordinate spaces when dealing with Kinect camera.

### World Space

![World coordinate definition.](https://i-msdn.sec.s-msft.com/dynimg/IC757720.png)

Addressed as "camera space" in the official document, it is actually the 3D world space.

*   **Origin**: Center of the IR sensor (Not center of the color camera)
*   **Coordinate frame**: Right-handed. y axis grows up, z axis grows out towards objects.
*   **Unit**: Meter (Not millimeter)

### Depth Space

Space of depth image. Each pixel on depth image has a depth value.

*   **Origin**: Left-top corner
*   **Coordinate axes**: x axis grows to the right; y axis grows down
*   **Range**: Resolution is $512 \times 424$. Therefore $x \in [0, 511]$, $ y \in [0, 423]$
*   **Depth value**: Depth value of each pixel is in millimeters. Valid depth range is 500 mm to 4500 mm. API retrieved array are row-major stored.

### Color Space

Space of color image. Each pixel on color image has a color value.

*   **Origin**: Left-top corner
*   **Coordinate axes**: x axis grows to the right; y axis grows down
*   **Range**: Resolution $1920 \times 1080$ (full HD 1080p). Therefore $x \in [0, 1919]$, $y \in [0, 1079]$
*   **Color value**: The color value directly retrieved from API is stored in `BGRX` format. Four bytes for each pixel, and the X channel is reserved and always `0xff`. The data array are row-major stored, which can be directly mapped to OpenCV `cv::Mat` with `CV_8UC4` format.

> **Pitfall 1. **
> The image in depth space and color space are different from common camera: they are horizontally mirrored. That is, if using a simple pin-hole camera matrix to model, the 2nd row of camera matrix should be negative.

## Coordinate Mapping

### Using API

Coordinates mapping can be done using [member methods of `ICoordinateMapper` class](https://msdn.microsoft.com/en-us/library/microsoft.kinect.kinect.icoordinatemapper.aspx) in the SDK. You can map one point, several points or the whole frame. But not all these three types of mappings between every two spaces are available. Specifically, the available mappings are illustrated as below:

![Kinect API-provided coordinate conversions](http://on02twadg.bkt.clouddn.com/kinect-api-coord-conv.svg)

> **Pitfall 2.** 
> Mapping between depth space and color space is related to the depth value. Therefore, if you want to filter depth values, please do it before mapping.

### Using Camera Matrix

The API functions are not easy to use when you want to build an optimization problem and camera matrix is assembled into your equation. In such cases, you may want to use camera matrix directly.

Note that you should use depth camera to project 3D points or back-project 2D pixels. Here are the reasons:

1.  Accurate intrinsic camera parameters of depth camera can be retrieved from SDK API. But it cannot be retrieved for color camera.
2.  Depth camera is located at the center of the world coordinate. This will make things easier if you are going to cooperate with other features of the SDK and stick to the defined world coordinate system.

With known camera matrix $P$,
$$P =
\begin{bmatrix}
f_x & & c_x\\\\
& f_y & c_y\\\\
& & 1
\end{bmatrix}$$

projecting a 3D point $\mathbf{v} = (x, y, z)^\top$ to depth space to get a 2D point $\mathbf{u} = (u,v)^\top$ with depth value $d$ is quite easy, because depth value is exactly $z$ value, i.e. $d = z$. Therefore,
$$
\begin{bmatrix}
f_x & & c_x\\\\
& f_y & c_y\\\\
& & 1
\end{bmatrix}
\begin{bmatrix}
x \\\\ y \\\\ z
\end{bmatrix}
=
\begin{bmatrix}
ud\\\\ vd\\\\ d
\end{bmatrix}
$$
It's not hard to get

*   Projection (3D to 2D)
$$\begin{array}{rl}
u & = \dfrac{f_x x + c_x z}{z}\\\\
v & = \dfrac{f_y y + c_y z}{z}\\\\
d & = z
\end{array}$$

*   Back-projection (2D to 3D)
$$\begin{array}{rl}
x &= \dfrac{(u - c_x) d}{f_x} \\\\
y &= \dfrac{(v - c_y) d}{f_y} \\\\
z &= d
\end{array}$$

> **Pitfall 3.** 
> The camera matrix above ignores radial distortion, while mapping functions in the SDK API don't. So you may want to be consistent with projection and back-projection: Use matrix for both or not at all.