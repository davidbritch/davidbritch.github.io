---
title: Animating augmented reality content in .NET MAUI on iOS
date: 2025-09-04
excerpt: Nodes in an augmented reality app can be animated with the `SCNAction` type, which represents a reusable animation that changes attributes of any node you attach it to. `SCNAction` objects are created with specific methods, and are executed by calling a node object's `RunAction` method, passing the action object as an argument.
tags: 
- "ARKit"
- ".NET MAUI"
---

<a href="https://github.com/davidbritch/augmented-reality-demos/tree/main/03%20-%20Rotating%20earth" class="btn btn--info">Download the code</a>

[Previously]({% post_url 2025-09-02-interacting-with-augmented-reality-content-in-net-maui %}) I discussed how to interact with augmented reality content in a .NET MAUI app on iOS. This involved creating gesture recognizers and adding them to the `ARSCNView` object with the `AddGestureRecognizer` method.

In this blog post I'll build on this to discuss how to animate the content you add to a scene, while still enabling it to respond to touch interaction. Specifically, I'll add a spherical image of the earth to the scene, and animate it by rotating it continuously through 360 degrees on the Y-axis.

**Note:** You'll need a physical iPhone to run an augmented reality app. ARKit requires the use of the camera, and you won't have much joy using an iOS simulator.
{: .notice--info}

## Animate a node

Objects that you overlay on the camera output are called *nodes*. By default, nodes don’t have a shape. Instead, you give them a geometry and apply materials to the geometry to provide a visual appearance. Nodes are represented by the SceneKit `SCNode` type.

One of the geometries provided by SceneKit is `SCNSphere`, which represents a sphere. In order to add a sphere to the scene, I've created a `SphereNode` type that derives from `SCNNode`:

```csharp
using SceneKit;
using UIKit;

namespace ARKitDemo.Platforms.iOS;

public class SphereNode : SCNNode
{
    public SphereNode(UIImage? image, float size)
    {
        var node = new SCNNode
        {
            Geometry = CreateGeometry(image, size),
            Opacity = 0.975f
        };

        AddChildNode(node);
    }

    SCNGeometry CreateGeometry(UIImage? image, float size)
    {
        SCNMaterial material = new SCNMaterial();
        material.Diffuse.Contents = image;
        material.DoubleSided = true;

        SCNSphere geometry = SCNSphere.Create(size);
        geometry.Materials = new[] { material };

        return geometry;
    }
}
```

The `SphereNode` constructor takes a `UIImage` argument that represents the image to be added to the sphere, and a `float` argument that represents the size of the sphere. The constructor creates the material and geometry for the sphere, and adds the node as a child node to the `SCNNode`. ARKit maps the image to the geometry of the sphere as a material. The result is that a 2D rectangular image (in this case a map of the world) is automatically wrapped around the sphere.

Nodes can be animated with the `SCNAction` type, which represents a reusable animation that changes attributes of any node you attach it to. `SCNAction` objects are created with specific methods, and are executed by calling a node object's `RunAction` method, passing the action object as an argument. For example, the following code creates a rotation action and applies it to a `SphereNode`:

```csharp
SCNAction rotateAction = SCNAction.RotateBy(0, (float)Math.PI, 0, 5); // X,Y,Z,secs

SphereNode sphereNode = new SphereNode(image, 0.1f);
sphereNode.RunAction(rotateAction);
sceneView.Scene.RootNode.AddChildNode(sphereNode);
```

In this example, the `SphereNode` is rotated 360 degrees on the Y-axis over 5 secsonds, meaning that it takes 5 seconds to complete a full 360 degree rotation. To rotate the sphere indefinitely, use the following code:

```csharp
SCNAction rotateAction = SCNAction.RotateBy(0, (float)Math.PI, 0, 5); // X,Y,Z,secs
SCNAction indefiniteRotation = SCNAction.RepeatActionForever(rotateAction);

SphereNode sphereNode = new SphereNode(image, 0.1f);
sphereNode.RunAction(indefiniteRotation);
sceneView.Scene.RootNode.AddChildNode(sphereNode);
```

In this example, the `SphereNode` is rotated 360 degrees on the Y-axis over 5 seconds. Then the animation is looped.

This rotation code can be generalised into an extension method that can be called on any `SCNNode` type:

```csharp
using SceneKit;

namespace ARKitDemo.Platforms.iOS;

public static class SCNNodeExtensions
{
    public static void AddRotationAction(this SCNNode node, SCNActionTimingMode mode, double secs, bool loop = false)
    {
        SCNAction rotateAction = SCNAction.RotateBy(0, (float)Math.PI, 0, secs);
        rotateAction.TimingMode = mode;

        if (loop)
        {
            SCNAction indefiniteRotation = SCNAction.RepeatActionForever(rotateAction);
            node.RunAction(indefiniteRotation, "rotation");
        }
        else
            node.RunAction(rotateAction, "rotation");
    }
}
```

The `AddRotationAction` extension method adds a rotate animation on the Y-axis to the specified `SCNNode`. The `SCNActionTimingMode` argument defines the easing function for the animation. The `secs` argument defines the number of seconds to complete a full rotation of the node, and the `loop` argument defines whether to animate the node indefinitely. The `RunAction` method calls both specify a `string` key argument. This enables the animation to be stopped programmatically by specifying the `key` as an argument to the `RemoveAction` method.

The `AddContent` method in the `MauiARView` class can then be modified to add a `SphereNode` to the scene:

```csharp
void AddContent()
{
    UIImage? image = UIImage.FromFile("worldmap.jpg");
    _node = new SphereNode(image, 0.005f);
    _arView?.Scene.RootNode.AddChildNode(_node);
}
```

In this example, the `AddContent` method loads a specified image from the app bundle into a `UIImage`. The `SphereNode` is then created from the image, and added to the scene at the world origin (0,0,0).

The `SphereNode` is animated through the use of a tap gesture:

```csharp
void HandleTapGesture(UITapGestureRecognizer? sender)
{
    SCNView? areaTapped = sender?.View as SCNView;
    CGPoint? location = sender?.LocationInView(areaTapped);
    SCNHitTestResult[]? hitTestResults = areaTapped?.HitTest((CGPoint)location!, new SCNHitTestOptions());
    SCNHitTestResult? hitTest = hitTestResults?.FirstOrDefault();

    SCNNode? node = hitTest?.Node;
    if (!_isAnimating)
    {
        node?.AddRotationAction(SCNActionTimingMode.Linear, 10, true);
        _isAnimating = true;
    }
    else
    {
        node?.RemoveAction("rotation");
        _isAnimating = false;
    }
}
```

In this example, the `HandleTapGesture` method determines the node on which the tap gesture is detected. If the node isn't being animated, an indefinite rotation `SCNAction` is added to the node, which fully rotates it every 10 seconds. Then, provided that the node is being animated, when it's tapped again the animation ceases by calling the `RemoveAction` method, specifying the key value for the action.

In addition to the tap gesture recognizer, the code also supports a pinch gesture to scale the size of the sphere, and a rotate gesture to rotate the sphere on it's Z-axis:

```csharp
void HandleRotateGesture(UIRotationGestureRecognizer? sender)
{
    SCNView? areaPinched = sender?.View as SCNView;
    CGPoint? location = sender?.LocationInView(areaPinched);
    SCNHitTestResult[]? hitTestResults = areaPinched?.HitTest((CGPoint)location!, new SCNHitTestOptions());
    SCNHitTestResult? hitTest = hitTestResults?.FirstOrDefault();

    if (hitTest == null)
        return;

    SCNNode node = hitTest.Node;
    _zAngle += (float)-sender!.Rotation;
    node.EulerAngles = new SCNVector3(node.EulerAngles.X, node.EulerAngles.Y, _zAngle);
}
```

In this example, `HandleRotateGesture` determines the node on which the rotate gesture is detected. Then the node is rotated on the Z-axis by the rotation angle requested by the gesture.

The overall effect is that when the AR session starts, a `SphereNode` that resembles the earth appears:

![](/assets/images/arkit-earth.png){: .align-center}

Tapping on the `SphereNode` starts it rotating, and while rotating, tapping it again stops it rotating. In addition, the pinch gesture will resize the `SphereNode`, and the rotate gesture enables the Z-axis of the `SphereNode` to be manipulated.