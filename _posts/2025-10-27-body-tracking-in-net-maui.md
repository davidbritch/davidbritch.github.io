---
title: Body tracking in .NET MAUI on iOS
date: 2025-10-28
excerpt: ARKit includes the ability to perform real-time body tracking in a scene, including the orientation of different body parts.
tags: 
- "ARKit"
- ".NET MAUI"
category: software
---

<a href="https://github.com/davidbritch/augmented-reality-demos/tree/main/06%20-%20Body%20tracking" class="btn btn--info">Download the code</a>

[Previously]({% post_url 2025-10-21-facial-expression-identification-in-net-maui %}) I discussed how to perform facial expression identification. ARKIt enables you to detect over 50 different facial expressions including blinking, squinting, smiling, and sneering, and can also identifiy multiple expressions simultaneously.

In this blog post I'll explore how to perform body tracking in a scene, including the orientation of different body parts. 

**Note:** You'll need a physical iPhone to run an augmented reality app. ARKit requires the use of the camera, and you won't have much joy using an iOS simulator.
{: .notice--info}

## Detect and track a body

When ARKit recognizes a body, it automatically adds a body anchor object (`ARBodyAnchor`) to the running session, from which you can track the body's movement. It's common practice to create a dedicated class, known as a scene view delegate (a class that derives from `ARSCNViewDelegate`), to handle the overrides that are called when an anchor is detected and place in the scene. To do this, you'll first have to tell your `ARSCNView` object about your scene view delegate. This can be achieved by modifying the `CreatePlatformView` method in your handler to set the `ARSCNView.Delegate` property to the object that represents your scene view delegate:

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

The scene view delegate class derives from `ARSCNViewDelegate`, and overrides specific methods that are called when a body anchor is detected and placed in the scene, or updated:

```csharp
using ARKit;
using CoreGraphics;
using Foundation;
using SceneKit;
using UIKit;

namespace ARKitDemo.Platforms.iOS;

public class SceneViewDelegate : ARSCNViewDelegate
{
    Dictionary<string, JointNode> _joints = new Dictionary<string, JointNode>();

    public override void DidAddNode(ISCNSceneRenderer renderer, SCNNode node, ARAnchor anchor)
    {
        if (!(anchor is ARBodyAnchor bodyAnchor))
            return;

        foreach (var jointName in ARSkeletonDefinition.DefaultBody3DSkeletonDefinition.JointNames)
        {
            JointNode jointNode = MakeJoint(jointName);
            var jointPosition = GetJointPosition(bodyAnchor, jointName);
            jointNode.Position = jointPosition;

            if (!_joints.ContainsKey(jointName))
            {
                node.AddChildNode(jointNode);
                _joints.Add(jointName, jointNode);
            }
        }
    }

    public override void DidUpdateNode(ISCNSceneRenderer renderer, SCNNode node, ARAnchor anchor)
    {
        if (!(anchor is ARBodyAnchor bodyAnchor))
            return;

        foreach (var jointName in ARSkeletonDefinition.DefaultBody3DSkeletonDefinition.JointNames)
        {
            var jointPosition = GetJointPosition(bodyAnchor, jointName);

            if (!_joints.ContainsKey(jointName))
                _joints[jointName].Update(jointPosition);
        }            
    }

    SCNVector3 GetJointPosition(ARBodyAnchor bodyAnchor, string jointName)
    {
        NMatrix4 jointTransform = bodyAnchor.Skeleton.GetModelTransform((NSString)jointName);
        return new SCNVector3(jointTransform.Column3);
    }

    UIColor GetJointColour(string jointName)
    {
        switch (jointName)
        {
            case "root":
            case "left_foot_joint":
            case "right_foot_joint":
            case "left_leg_joint":
            case "right_leg_joint":
            case "left_hand_joint":
            case "right_hand_joint":
            case "left_arm_joint":
            case "right_arm_joint":
            case "left_forearm_joint":
            case "right_forearm_joint":
            case "head_joint":
                return UIColor.Red;
        }

        return UIColor.Gray;
    }

    float GetJointRadius(string jointName)
    {
        switch (jointName)
        {
            case "root":
            case "left_foot_joint":
            case "right_foot_joint":
            case "left_leg_joint":
            case "right_leg_joint":
            case "left_hand_joint":
            case "right_hand_joint":
            case "left_arm_joint":
            case "right_arm_joint":
            case "left_forearm_joint":
            case "right_forearm_joint":
            case "head_joint":
                return 0.04f;
        }

        if (jointName.Contains("hand"))
            return 0.01f;

        return 0.02f;
    }

    JointNode MakeJoint(string jointName)
    {
        var jointNode = new JointNode();
        var material = new SCNMaterial();
        material.Diffuse.Contents = GetJointColour(jointName);

        var jointGeometry = SCNSphere.Create(GetJointRadius(jointName));
        jointGeometry.FirstMaterial = material;
        jointNode.Geometry = jointGeometry;

        return jointNode;
    }
}
```

In this example, the `DidAddNode` method is called when a node (a new body) is detected in the scene. This method uses a `JointNode` class, which inherits from `SCNNode`, to represent the joint nodes to be placed in the scene. These joint nodes are stored in a `Dictionary`, using the joint name as the key. Each joint is represented by a sphere, that's created by the `MakeJoint` method. The `GetJointPosition` takes the `ARBodyAnchor` object and calculates and returns the position of the joint. This is achieved by determining the joints offset from the root position of the `ARBodyAnchor`, which is always the center of the hip.

The `DidUpdateNode` method is called when the body position changes, which updates the joint position with the `Update` method.

Body tracking differs from other uses of ARKit in the class you use to configure the session. To enable body tracking, create an `ARBodyTrackingConfiguration` object and configure its properties. This is achieved in the `StartARSession` method in the `MauiARView` class:

```csharp
public class MauiARView : UIView
{
    ARSession? _arSession;
    ...

    public void StartARSession()
    {
        if (_arSession == null)
            return;

        _arSession.Run(new ARBodyTrackingConfiguration()
        {
            WorldAlignment = ARWorldAlignment.Gravity
        });
    }
    ...
}
```

The `ARBodyTrackingConfiguration` uses the rear camera by default, enabling you to detect and track a body in the scene. Major joints are shown as red spheres, and minor joints are shown as grey nodes.
