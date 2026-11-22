---
title: Changing the Webcam Box Shape in OBS Studio
description: Crop and mask a webcam source in OBS Studio to create a circle, rounded square or custom shape, with optional borders and troubleshooting tips.
weight: 1
categories: [tutorial]
tags: [obs-studio, webcam, streaming, video, windows, macos, linux]
---

## Overview

Give your talking-head webcam a circular, rounded or custom-shaped frame using OBS Studio's built-in crop and image mask filters. This works on Windows, macOS and Linux; camera source names and keyboard shortcuts vary by platform.

## Before You Start

- Install [OBS Studio](https://obsproject.com/) and connect your webcam.
- Have an image editor available, such as GIMP, Krita, Photoshop or Photopea, to create a PNG mask.

## 1. Add Your Webcam

If your webcam is already in the scene, skip to the next step.

1. In the **Sources** dock, click **+**.
2. Choose the camera source for your operating system:
   - **Windows:** Video Capture Device
   - **macOS:** Video Capture Device (called macOS Video Capture Device in some versions)
   - **Linux:** Video Capture Device (V4L2)
3. Select your camera and click **OK**.

## 2. Crop the Webcam to a Square

For a circular or square frame, crop the camera to a square before applying the mask. Many webcams output a widescreen image, which can otherwise stretch a circle into an oval.

1. Right-click the webcam source and choose **Filters**.
2. Under **Effect Filters**, click **+** and add **Crop/Pad**.
3. Leave **Relative** checked so you can enter the number of pixels to remove from each edge.
4. For a 1920 × 1080 camera, set **Left** and **Right** to **420**, leaving **Top** and **Bottom** at **0**. The result is 1080 × 1080.
5. Adjust **Left** and **Right** to centre your face, keeping their total at **840**.

For a different resolution, subtract the height from the width and split the difference between the left and right edges. Check your camera's actual output resolution first.

{{% callout tip %}}
The Crop/Pad filter affects this source everywhere it is used. For a simple rectangular crop in one scene, select the source on the canvas, hold **Alt** on Windows or Linux, or **Option** on macOS, and drag an edge handle inward. Canvas cropping happens after filters, so use Crop/Pad before the mask for the square example above.
{{% /callout %}}

## 3. Make a Mask Image

1. In your image editor, create a square canvas matching the cropped source, such as **1080 × 1080 px**.
2. Fill the background with **black**.
3. Draw the desired shape in **white**:
   - **Circle:** draw an ellipse filling the canvas.
   - **Rounded square:** draw a rectangle with rounded corners.
   - **Custom shape:** draw a hexagon, star or any other shape.
4. Export the image as a **PNG**, with anti-aliasing enabled for smooth edges.

For a colour-channel mask, white areas are visible and black areas are hidden. Grey edge pixels create partial transparency.

## 4. Apply the Mask in OBS

1. Right-click the webcam source and choose **Filters**.
2. Under **Effect Filters**, click **+** and add **Image Mask/Blend**.
3. Set **Type** to **Alpha Mask (Color Channel)** and leave **Color** white.
4. Click **Browse** and select your mask PNG.
5. Make sure **Crop/Pad** is above **Image Mask/Blend** in the effect filter list, so the mask is applied to the cropped image.

The webcam should now appear only inside your shape. If your PNG has an opaque shape on a transparent background instead, choose **Alpha Mask (Alpha Channel)**: opaque pixels remain visible and transparent pixels are hidden.

## 5. Position and Resize

- Drag the webcam on the canvas to move it.
- Drag a corner handle to resize it while keeping its proportions. Hold **Shift** while dragging if you want to stretch it.
- For precise positioning, right-click the source and choose **Transform → Edit Transform**.

## Simple Rectangles

For a 4:3, portrait or other rectangular box, the **Crop/Pad** filter is enough; you do not need a mask. You can also crop a single scene's source by dragging an edge while holding **Alt** on Windows or Linux, or **Option** on macOS.

## Rounded Corners with a Plugin

If you prefer not to create a mask, a shader plugin can provide rounded corners:

- [obs-shaderfilter](https://github.com/exeldro/obs-shaderfilter) adds a **User-defined shader** filter and includes a `rounded_rect.shader` example.
- [StreamFX](https://github.com/Xaymar/obs-StreamFX) offers shader and mask options. Check its releases for operating-system and OBS version compatibility.

Follow the plugin's installation instructions for your operating system, then restart OBS. Available builds and features depend on the plugin version.

## Add a Border

1. Make a second PNG with a coloured ring matching the mask shape. Leave the middle and outside transparent.
2. Add the PNG as an **Image** source above the webcam in the **Sources** list.
3. Position and resize it to line up with the webcam.

## Troubleshooting

| Problem | Fix |
|---|---|
| Circle looks like an oval | Check that Crop/Pad produces a square, the mask is square, and the source has not been stretched on the canvas. |
| Shape is inverted | For a colour-channel mask, swap the black and white areas. For an alpha-channel mask, make the desired shape opaque and the outside transparent. |
| Whole webcam disappears | Check the mask file and filter type. Use **Alpha Mask (Alpha Channel)** for a transparent-background mask, or **Alpha Mask (Color Channel)** for a black-and-white mask. |
| Mask edges look jagged | Use an anti-aliased mask at the source resolution or larger. |
| Mask no longer lines up after changing camera resolution | Recalculate the crop for the new resolution and check the mask's aspect ratio and filter order. |
| Camera not listed on macOS | Allow OBS access under **System Settings → Privacy & Security → Camera**, then restart OBS. |
| Camera not listed on Linux | Check that the camera appears under `/dev/video*` and that your user has permission to access it; the required permissions vary by distribution. |

For more details, see the OBS documentation for the [Crop/Pad](https://obsproject.com/kb/crop-pad-filter) and [Image Mask/Blend](https://obsproject.com/kb/image-mask-blend-filter) filters.
