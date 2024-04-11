---
tags: 
  - unreal
title: "UI Material: Radial Fade Transition with Offset"
categories:
  - Materials
---

<img src="https://img.shields.io/badge/Unreal%20Engine-informational" alt="Written for Unreal Engine"> <img src="https://img.shields.io/badge/-Materials-teal" alt="Materials">

## Preview
<img src="/assets/images/radial-fade-preview.gif" width="50%">

## Complete material
Click for full resolution.
<a href="https://unrealist.org/assets/images/complete-material.png" target="_blank"><img src="/assets/images/complete-material.png"></a>

|Parameter Name|Type|
|---------|----|
|`Texture`|Texture Sample|
|`Value`|Scalar 0.0–1.0|
|`CenterX`|Scalar 0.0–1.0|
|`CenterY`|Scalar 0.0–1.0|

## Usage
Put your widget in a *Retainer Box* and set the Effect Material.

<img src="/assets/images/radial-fade-retainer-box-details.png">

Then on each tick, update the *Value* parameter to a number between 0–1.

In my game, I evaluate a curve and then plug the value into the effect material's parameter named *Value*.

<img src="/assets/images/radial-fade-usage.png">

## How it works
Let's begin with a simple radial gradient which can be accomplished with the *Distance* node. We want to use the normalized UV of **GetUserInterfaceUV** because it'll span the entire widget even when the brush is being drawn as a box or border. This is mapped to texture coordinate index 4.

<img src="/assets/images/radial-fade-1.png">

For this material, we want to control the fade transition with a scalar parameter named *Value*. Our goal is to make it so that a value of 0 makes the widget completely invisible, and a value of 1 makes it completely visible.

By subtracting the distance from the parameter, we move step closer to achieving this.

<img src="/assets/images/radial-fade-2.png">

The result of the *Subtract* node will always be negative when *Value* is set to 0, rendering the widget completely invisible. As *Value* approaches 1, UVs closer to the center point will begin to output a positive number causing the widget to begin revealing itself outwards from the center point.

<img src="/assets/images/radial-fade-gif-1.gif">

There are still two issues here:

1. *Value* is clamped to 1 so subtraction will never output 1 beyond the center which always produces a visible gradient.
2. Distances greater than 1, which happens when the center point is shifted to a corner, always output negative values and this portion of the texture will never be visible.

To solve this, let's multiply the distance with *Value* and then add it to the output of the subtraction.

<img src="/assets/images/radial-fade-3.png">

When *Value* is 1, both outputs negate each other and provides us with a uniform value of 1.0 for all UVs — no matter where the center point is.

<img src="/assets/images/radial-fade-gif-3.gif">
<img src="/assets/images/radial-fade-gif-2.gif">

## Conclusion
There's a surprising lack of information on materials out there, especially User Interface materials. It was challenging for me to learn how to create shaders, so I'm sharing my materials and explaining how they work in hopes that it will help others. :)
