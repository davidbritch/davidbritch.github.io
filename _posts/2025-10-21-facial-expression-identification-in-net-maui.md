---
title: Facial expression identification in .NET MAUI on iOS
date: 2025-10-21
excerpt: ARKIt enables you to detect over 50 different facial expressions including blinking, squinting, smiling, and sneering, and can also identifiy multiple expressions simultaneously.
tags: 
- "ARKit"
- ".NET MAUI"
category: software
---

<a href="https://github.com/davidbritch/augmented-reality-demos/tree/main/05%20-%20Face%20expressions" class="btn btn--info">Download the code</a>

[Previously]({% post_url 2025-10-13-face-detection-in-net-maui %}) I discussed how to detect and track a face in a scene. This involved configuring and running a face tracking session with ARKit.

In this blog post I'll explore how to perform facial expression identification. ARKIt enables you to detect over 50 different facial expressions including blinking, squinting, smiling, and sneering, and can also identifiy multiple expressions simultaneously.

**Note:** You'll need a physical iPhone to run an augmented reality app. ARKit requires the use of the camera, and you won't have much joy using an iOS simulator.
{: .notice--info}

## Identify a facial expression

Just as with face detection, you'll need to configure and run a face tracking session with ARKit, with a scene view delegate being the place to add code to perform facial expression identification. For more information about this configuration, see my [previous]({% post_url 2025-10-13-face-detection-in-net-maui %}) blog post.

The scene view delegate class derives from `ARSCNViewDelegate`, and overrides specific methods that are called when face anchors are detected and placed in the scene, or updated:

```csharp
using ARKit;
using SceneKit;
using UIKit;

namespace ARKitDemo.Platforms.iOS;

public class SceneViewDelegate : ARSCNViewDelegate
{
    public override void DidAddNode(ISCNSceneRenderer renderer, SCNNode node, ARAnchor anchor)
    {
        if (anchor is ARFaceAnchor)
        {
            ARSCNFaceGeometry? faceGeometry = ARSCNFaceGeometry.Create(renderer.Device!);
            node.Geometry = faceGeometry;
            node.Geometry.FirstMaterial.FillMode = SCNFillMode.Fill;
        }
    }

    public override void DidUpdateNode(ISCNSceneRenderer renderer, SCNNode node, ARAnchor anchor)
    {
        if (anchor is ARFaceAnchor)
        {
            ARFaceAnchor? faceAnchor = anchor as ARFaceAnchor;
            ARSCNFaceGeometry? faceGeometry = node.Geometry as ARSCNFaceGeometry;
            float expressionThreshold = 0.5f;

            faceGeometry?.Update(faceAnchor!.Geometry);

            if (faceAnchor?.BlendShapes.EyeWideLeft > expressionThreshold
                || faceAnchor?.BlendShapes.EyeWideRight > expressionThreshold)
            {
                ChangeFaceColour(node, UIColor.Blue);
                return;
            }

            if (faceAnchor?.BlendShapes.EyeBlinkLeft > expressionThreshold
                || faceAnchor?.BlendShapes.EyeBlinkRight > expressionThreshold)
            {
                ChangeFaceColour(node, UIColor.Green);
                return;
            }

            if (faceAnchor?.BlendShapes.MouthFrownLeft > expressionThreshold
                || faceAnchor?.BlendShapes.MouthFrownRight > expressionThreshold)
            {
                ChangeFaceColour(node, UIColor.SystemPink);
                return;
            }

            if (faceAnchor?.BlendShapes.MouthSmileLeft > expressionThreshold
                || faceAnchor?.BlendShapes.MouthSmileRight > expressionThreshold)
            {
                ChangeFaceColour(node, UIColor.Black);
                return;
            }

            if (faceAnchor?.BlendShapes.BrowOuterUpLeft > expressionThreshold
                || faceAnchor?.BlendShapes.BrowOuterUpRight > expressionThreshold)
            {
                ChangeFaceColour(node, UIColor.Magenta);
                return;
            }

            if (faceAnchor?.BlendShapes.EyeLookOutLeft > expressionThreshold
                || faceAnchor?.BlendShapes.EyeLookOutRight > expressionThreshold)
            {
                ChangeFaceColour(node, UIColor.Cyan);
                return;
            }

            if (faceAnchor?.BlendShapes.TongueOut > expressionThreshold)
            {
                ChangeFaceColour(node, UIColor.Yellow);
                return;
            }

            if (faceAnchor?.BlendShapes.MouthFunnel > expressionThreshold)
            {
                ChangeFaceColour(node, UIColor.Red);
                return;
            }

            if (faceAnchor?.BlendShapes.CheekPuff > expressionThreshold)
            {
                ChangeFaceColour(node, UIColor.Orange);
                return;
            }

            ChangeFaceColour(node, UIColor.White);
        }
    }

    void ChangeFaceColour(SCNNode faceGeometry, UIColor colour)
    {
        SCNMaterial material = new SCNMaterial();
        material.Diffuse.Contents = colour;
        material.FillMode = SCNFillMode.Fill;

        faceGeometry.Geometry.FirstMaterial = material;
    }
}
```

ARKit provides a coarse 3D mesh geometry that matches the size, shape, topology, and current facial expression of a detected face. This mesh geometry is encapsulated by the `ARSCNFaceGeometry` class, which provides an easy way to visualise the mesh. In this example, the `DidAddNode` method is called when a node (a new face) is detected in the scene. This method creates the facial geometry and sets it to be the geometry of the node that's placed at the location of the `ARFaceAnchor`. 

ARKit provides the app with an updated anchor when the user changes their expression, and this is handled by the `DidUpdateNote` method. The app reads the user's current expression by interpreting the anchor's *blend shapes*, which are floating point values between 0 and 1 to denote the absence or presence of the expression. Alternatively, you can think of 0 as the facial feature's rest position, and 1 as the feature in its most pronounced state. For a full list of identifiable face expressions, see [ARFaceAnchor.BlendShapeLocation](https://developer.apple.com/documentation/arkit/arfaceanchor/blendshapelocation) in Apple's docs.

In response to a specific expression being detected (being greater than a threshold), the `DidUpdateNode` method changes the mesh geometry that's overlaid on the user's face to a specific colour:

![](/assets/images/arkit-expression-identification.png){: .align-center}

While changing the colour of the mesh geometry that's overlaid on the user's face isn't that useful, it does illustrate how straightforward it is to perform tasks such as gaze tracking and emotion recognition.
