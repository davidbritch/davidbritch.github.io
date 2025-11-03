---
title: Image detection in .NET MAUI on iOS
date: 2025-11-03
excerpt: ARKit can detect images in a scene by retrieving them from your app's asset catalog. Once you recognize images in a scene, you can perform additional tasks such as replacing them or otherwise augmenting them.
tags: 
- "ARKit"
- ".NET MAUI"
category: software
---

<a href="https://github.com/davidbritch/augmented-reality-demos/tree/main/07%20-%20Image%20detection" class="btn btn--info">Download the code</a>

[Previously]({% post_url 2025-10-27-body-tracking-in-net-maui %}) I discussed how to perform body tracking in a scene, including the orientation of different body parts. When ARKit recognizes a body, it automatically adds a body anchor object to the running session, from which you can track the body’s movement.

In this blog post, I'll discuss how to perform image detection in a scene. Once you recognize an image in a scene, you can perform additional tasks such as replacing it or otherwise augmenting it. Specifically, the app will identify the following image in a scene, and highlight it:

![](/assets/images/arkit-bookcover.jpg){: .align-center}

**Note:** You'll need a physical iPhone to run an augmented reality app. ARKit requires the use of the camera, and you won't have much joy using an iOS simulator.
{: .notice--info}

## Perform image detection

The simplest approach to declaring the image to be detected is to add it to your app's asset catalog as an **AR Reference Image** inside an **AR Resource Group**. It's also possible to place multiple images in the asset catalog, for identification in a scene.

When ARKit recognizes an image in the scene, it automatically adds an image anchor object (`ARImageAnchor`) to the running session. It's common practice to create a dedicated class, known as a scene view delegate (a class that derives from `ARSCNViewDelegate`), to handle the overrides that are called when an anchor is detected and place in the scene. To do this, you'll first have to tell your `ARSCNView` object about your scene view delegate. This can be achieved by modifying the `CreatePlatformView` method in your handler to set the `ARSCNView.Delegate` property to the object that represents your scene view delegate:

```csharp
public class ARViewHandler : ViewHandler<ARView, ARSCNView>
{
    MauiARView? _mauiARView;
    ...

    protected override ARSCNView CreatePlatformView()
    {
        var arView = new ARSCNView()
        {
            AutoenablesDefaultLighting = true,
            ShowsStatistics = true,
            Delegate = new SceneViewDelegate(),
        };

        _mauiARView = new MauiARView();
        _mauiARView.SetARView(arView);

        return arView;
    }
    ...
}
```

The scene view delegate class derives from `ARSCNViewDelegate`, and overrides the `DidAddNode` method that's called when an anchor is detected and placed in the scene:

```csharp
using ARKit;
using SceneKit;
using UIKit;

namespace ARKitDemo.Platforms.iOS;

public class SceneViewDelegate : ARSCNViewDelegate
{
    public override void DidAddNode(ISCNSceneRenderer renderer, SCNNode node, ARAnchor anchor)
    {
        if (anchor is ARImageAnchor imageAnchor)
        {
            ARReferenceImage image = imageAnchor.ReferenceImage;
            nfloat width = image.PhysicalSize.Width;
            nfloat height = image.PhysicalSize.Height;

            PlaneNode planeNode = new PlaneNode(width, height, new SCNVector3(0, 0, 0), UIColor.Red);
            float angle = (float)-Math.PI / 2;
            planeNode.EulerAngles = new SCNVector3(angle, 0, 0);
            node.AddChildNode(planeNode);
        }
    }
}
```

The `SceneViewDelegate` class overrides the `DidAddNode` method, which is executed when the image is detected in the scene. This method first checks that the detected image is an `ARImageAnchor`, which represents an anchor for a known image that ARKit detects in the scene. Then the dimensions of the detected image are determined, and a red `PlaneNode` (of the same dimensions) is created and overlaid on the detected image. In addition, the overlaid `PlaneNode` will always orient itself correctly over the detected image.

The `PlaneNode` class is simply an `SCNNode`, which uses an `SCNPlane` geometry that represents a square or rectangle:

```csharp
using SceneKit;
using UIKit;

namespace ARKitDemo.Platforms.iOS;

public class PlaneNode : SCNNode
{
    public PlaneNode(nfloat width, nfloat length, SCNVector3 position, UIColor color)
    {
        SCNNode node = new SCNNode
        {
            Geometry = CreateGeometry(width, length, color),
            Position = position,
            Opacity = 0.5f
        };

        AddChildNode(node);
    }

    SCNGeometry CreateGeometry(nfloat width, nfloat length, UIColor color)
    {
        SCNMaterial material = new SCNMaterial();
        material.Diffuse.Contents = color;
        material.DoubleSided = false;

        SCNPlane geometry = SCNPlane.Create(width, length);
        geometry.Materials = new[] { material };

        return geometry;
    }    
}
```

The `PlaneNode` constructor takes arguments that represent the width and height of the node, it’s position, and a color. The constructor creates a `SCNNode`, assigns a geometry to its `Geometry` property, sets its position and opacity, and adds the child node to the `SCNNode`.

To enable body tracking, create an `ARWorldTrackingConfiguration` object and configure its properties. This is achieved in the `StartARSession` method in the `MauiARView` class:

```csharp
public void StartARSession()
{
    if (_arSession == null)
        return;

    NSSet<ARReferenceImage>? images = ARReferenceImage.GetReferenceImagesInGroup("AR Resources", null);

    _arSession.Run(new ARWorldTrackingConfiguration()
    {
        AutoFocusEnabled = true,
        LightEstimationEnabled = true,
        DetectionImages = images
    }, ARSessionRunOptions.ResetTracking | ARSessionRunOptions.RemoveExistingAnchors);
}
```

In addition, the `StartARSession` method retrieves the image to be detected from the asset catalog, and sets it as the `DetectionImages` property of the `ARWorldTrackingConfiguration` object.

The overall effect is that when the image is detected in the scene, a red rectangle is overlaid on it:

![](/assets/images/arkit-image-detection.png){: .align-center}

Then, the red rectangle reorients itself in real time if the orientation of the detected image in the scene changes.

Once you recognize an image in a scene, you can perform additional tasks such as replacing it or otherwise augmenting it.
