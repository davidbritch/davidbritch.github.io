---
title: Path flattening in SkiaSharp
date: 2025-08-11
excerpt: It's also sometimes useful to obtain all the drawing operations and points that make up a path, particularly for paths created by path effects and converting text strings into paths.
tags: 
- "SkiaSharp"
- ".NET MAUI"
category: software
---

<a href="https://github.com/davidbritch/skiasharp-demos/tree/main/PathFlattening" class="btn btn--info">Download the code</a>

The `SKPath` class defines several properties and methods that allow you to obtain information about a path. The `Bounds` and `TighBounds` properties, and related methods, obtain the metrical dimensions of a path. The `Contains` method lets you determine if a particular point is within a path.

It's also sometimes useful to obtain all the drawing operations and points that make up a path. This might seem unnecessary. After all, if your code has created the path, you should already know the contents. However, paths can also be created by path effects and converting text strings into paths. The good news is that it's also possible to obtain all the drawing operations and points that make up these paths with a technique known as *path flattening*. This technique was taught to me by the great [Charles Petzold](https://www.charlespetzold.com) many years ago, and is particularly useful for enabling scenarios where path effects don't provide a solution.

For example, consider the following example in which all the points in a path have been transformed to wrap text around a hemisphere:

![](/assets/images/skiasharp-globe.png){: .align-center}

In this example, most of the letters consist of straight lines that have been twisted into curves. This is achieved by breaking the original straight lines into a series of smaller straight lines, which are manipulated to form curves. This is the purpose of the `Interpolate` method in the custom [`PathExtensions`](https://github.com/davidbritch/skiasharp-demos/blob/main/PathFlattening/PathExtensions.cs) class, which breaks down a straight line into numerous short lines that are only one unit in length. In addition, the class contains several methods that convert cubic, quadratic, and conic Bezier curves into a series of tiny straight lines that approximate the curve, which is known as flattening the curve:

```csharp
using SkiaSharp;

namespace PathFlattening;

static class PathExtensions
{
    ...
    static SKPoint[] Interpolate(SKPoint pt0, SKPoint pt1)
    {
        int count = (int)Math.Max(1, Length(pt0, pt1));
        SKPoint[] points = new SKPoint[count];

        for (int i = 0; i < count; i++)
        {
            float t = (i + 1f) / count;
            float x = (1 - t) * pt0.X + t * pt1.X;
            float y = (1 - t) * pt0.Y + t * pt1.Y;
            points[i] = new SKPoint(x, y);
        }
        return points;
    }

    static SKPoint[] FlattenCubic(SKPoint pt0, SKPoint pt1, SKPoint pt2, SKPoint pt3)
    {
        int count = (int)Math.Max(1, Length(pt0, pt1) + Length(pt1, pt2) + Length(pt2, pt3));
        SKPoint[] points = new SKPoint[count];

        for (int i = 0; i < count; i++)
        {
            float t = (i + 1f) / count;
            float x = (1 - t) * (1 - t) * (1 - t) * pt0.X +
                      3 * t * (1 - t) * (1 - t) * pt1.X +
                      3 * t * t * (1 - t) * pt2.X +
                      t * t * t * pt3.X;
            float y = (1 - t) * (1 - t) * (1 - t) * pt0.Y +
                      3 * t * (1 - t) * (1 - t) * pt1.Y +
                      3 * t * t * (1 - t) * pt2.Y +
                      t * t * t * pt3.Y;
            points[i] = new SKPoint(x, y);
        }
        return points;
    }

    static SKPoint[] FlattenQuadratic(SKPoint pt0, SKPoint pt1, SKPoint pt2)
    {
        int count = (int)Math.Max(1, Length(pt0, pt1) + Length(pt1, pt2));
        SKPoint[] points = new SKPoint[count];

        for (int i = 0; i < count; i++)
        {
            float t = (i + 1f) / count;
            float x = (1 - t) * (1 - t) * pt0.X + 2 * t * (1 - t) * pt1.X + t * t * pt2.X;
            float y = (1 - t) * (1 - t) * pt0.Y + 2 * t * (1 - t) * pt1.Y + t * t * pt2.Y;
            points[i] = new SKPoint(x, y);
        }
        return points;
    }

    static SKPoint[] FlattenConic(SKPoint pt0, SKPoint pt1, SKPoint pt2, float weight)
    {
        int count = (int)Math.Max(1, Length(pt0, pt1) + Length(pt1, pt2));
        SKPoint[] points = new SKPoint[count];

        for (int i = 0; i < count; i++)
        {
            float t = (i + 1f) / count;
            float denominator = (1 - t) * (1 - t) + 2 * weight * t * (1 - t) + t * t;
            float x = (1 - t) * (1 - t) * pt0.X + 2 * weight * t * (1 - t) * pt1.X + t * t * pt2.X;
            float y = (1 - t) * (1 - t) * pt0.Y + 2 * weight * t * (1 - t) * pt1.Y + t * t * pt2.Y;
            x /= denominator;
            y /= denominator;
            points[i] = new SKPoint(x, y);
        }
        return points;
    }

    static double Length(SKPoint pt0, SKPoint pt1)
    {
        return Math.Sqrt(Math.Pow(pt1.X - pt0.X, 2) + Math.Pow(pt1.Y - pt0.Y, 2));
    }    
}
```

All of these methods are referenced from the `CloneWithTransform` extension method, which is also in the custom [`PathExtensions`](https://github.com/davidbritch/skiasharp-demos/blob/main/PathFlattening/PathExtensions.cs) class. This extension method clones a path by enumerating the path commands and constructing a new path based on the data:

```csharp
using SkiaSharp;

namespace PathFlattening;

static class PathExtensions
{
    public static SKPath CloneWithTransform(this SKPath pathIn, Func<SKPoint, SKPoint> transform)
    {
        SKPath pathOut = new SKPath();

        using (SKPath.RawIterator iterator = pathIn.CreateRawIterator())
        {
            SKPoint[] points = new SKPoint[4];
            SKPathVerb pathVerb = SKPathVerb.Move;
            SKPoint firstPoint = new SKPoint();
            SKPoint lastPoint = new SKPoint();

            while ((pathVerb = iterator.Next(points)) != SKPathVerb.Done)
            {
                switch (pathVerb)
                {
                    case SKPathVerb.Move:
                        pathOut.MoveTo(transform(points[0]));
                        firstPoint = lastPoint = points[0];
                        break;

                    case SKPathVerb.Line:
                        SKPoint[] linePoints = Interpolate(points[0], points[1]);

                        foreach (SKPoint pt in linePoints)
                        {
                            pathOut.LineTo(transform(pt));
                        }

                        lastPoint = points[1];
                        break;

                    case SKPathVerb.Cubic:
                        SKPoint[] cubicPoints = FlattenCubic(points[0], points[1], points[2], points[3]);

                        foreach (SKPoint pt in cubicPoints)
                        {
                            pathOut.LineTo(transform(pt));
                        }

                        lastPoint = points[3];
                        break;

                    case SKPathVerb.Quad:
                        SKPoint[] quadPoints = FlattenQuadratic(points[0], points[1], points[2]);

                        foreach (SKPoint pt in quadPoints)
                        {
                            pathOut.LineTo(transform(pt));
                        }

                        lastPoint = points[2];
                        break;

                    case SKPathVerb.Conic:
                        SKPoint[] conicPoints = FlattenConic(points[0], points[1], points[2], iterator.ConicWeight());

                        foreach (SKPoint pt in conicPoints)
                        {
                            pathOut.LineTo(transform(pt));
                        }

                        lastPoint = points[2];
                        break;

                    case SKPathVerb.Close:
                        SKPoint[] closePoints = Interpolate(lastPoint, firstPoint);

                        foreach (SKPoint pt in closePoints)
                        {
                            pathOut.LineTo(transform(pt));
                        }

                        firstPoint = lastPoint = new SKPoint(0, 0);
                        pathOut.Close();
                        break;
                }
            }
        }
        return pathOut;
    }
    ...
}
```

The new path constructed by the `CloneWithTransform` extension method consists only of `MoveTo` and `LineTo` calls, and so all the curves and straight lines are reduced to a series of tiny lines. When calling `CloneWithTransform` you pass a `Func<SKPoint, SKPoint>` to the method, which is a function with an `SKPaint` parameter that returns an `SKPoint` value. This function is called for every point to apply a custom algorithmic transform. Because the cloned path is reduced to tiny lines, the transform function has the capability of converting straight lines to curves.

**Note:** The `CloneWithTransform` method retains the first point of each contour in the variable named `firstPoint` and the current position after each drawing command in the variable `lastPoint`. These variables are necessary to construct the final closing line with a `Close` operation.
{: .notice--info}

To return to the earlier example, the `CloneWithTransform` extension method can be called to give the appearance of text wrapped around a hemisphere:

![](/assets/images/skiasharp-globe.png){: .align-center}

This appearance can be achieved by creating an `SKFont` object for the text, and then obtaining an `SKPath` object from the `GetTextPath` method. This is the path that's passed to the `CloneWithTransform` extension method along with a transform function:

```csharp
using SkiaSharp.Views.Maui;
using SkiaSharp;

namespace PathFlattening;

public partial class MainPage : ContentPage
{
    SKPath globePath;

    public MainPage()
    {
        InitializeComponent();        

        using (SKFont font = new SKFont())
        {
            font.Typeface = SKTypeface.FromFamilyName("Times New Roman");
            font.Size = 100;

            using (SKPath textPath = font.GetTextPath("SKIASHARP", new SKPoint(0, 0)))
            {
                SKRect textPathBounds;
                textPath.GetBounds(out textPathBounds);

                globePath = textPath.CloneWithTransform((SKPoint pt) =>
                {
                    double longitude = (Math.PI / textPathBounds.Width) * (pt.X - textPathBounds.Left) - Math.PI / 2;
                    double latitude = (Math.PI / textPathBounds.Height) * (pt.Y - textPathBounds.Top) - Math.PI / 2;

                    longitude *= 0.75;
                    latitude *= 0.75;

                    float x = (float)(Math.Cos(latitude) * Math.Sin(longitude));
                    float y = (float)Math.Sin(latitude);

                    return new SKPoint(x, y);
                });
            }
        }
    }
    ...
}
```

The transform function first calculates two values named `longitude` and `latitude` that range from –π/2 at the top and left of the text, to π/2 at the right and bottom of the text. The range of these values isn't visually satisfactory, so they are reduced by multiplying by 0.75. These three-dimensional spherical coordinates are then converted to two-dimensional `x` and `y` coordinates.

The new path is stored in the `globePath` field. The `PaintSurface` event handler then just needs to center and scale the path to display it:

```csharp
using SkiaSharp.Views.Maui;
using SkiaSharp;

namespace PathFlattening;

public partial class MainPage : ContentPage
{
    SKPath globePath;

    ...
    void OnCanvasViewPaintSurface(object sender, SKPaintSurfaceEventArgs e)
    {
        SKImageInfo info = e.Info;
        SKSurface surface = e.Surface;
        SKCanvas canvas = surface.Canvas;

        canvas.Clear();

        using (SKPaint pathPaint = new SKPaint())
        {
            pathPaint.Style = SKPaintStyle.Fill;
            pathPaint.Color = SKColors.Blue;
            pathPaint.StrokeWidth = 3;
            pathPaint.IsAntialias = true;

            canvas.Translate(info.Width / 2, info.Height / 2);
            canvas.Scale(0.45f * Math.Min(info.Width, info.Height)); // Radius
            canvas.DrawPath(globePath, pathPaint);
        }        
    }
}
```
