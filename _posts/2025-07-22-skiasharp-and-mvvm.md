---
title: SkiaSharp and MVVM
date: 2025-07-22
tags:
- "SkiaSharp"
- ".NET MAUI"
---

[SkiaSharp](https://github.com/mono/SkiaSharp) is a 2D graphics system for .NET and C#, powered by Google's [Skia](https://skia.org) graphics engine. It's available as a NuGet package, and can easily be added to any .NET project, particularly .NET MAUI projects.

Back in the Xamarin.Forms days we had quite extensive [documentation](https://learn.microsoft.com/en-us/previous-versions/xamarin/xamarin-forms/user-interface/graphics/skiasharp/) for using SkiaSharp in Xamarin.Forms. However, while these docs are still live they aren't easily found as all the Xamarin docs have been forcibly de-indexed by search engines. Due to resource constraints, and the size of the SkiaSharp user base, the docs were never ported to .NET MAUI. However, the Xamarin.Forms SkiaSharp documentation still largely applies to .NET MAUI. It's just that sometimes APIs have been deprecated.

I'm not going to re-iterate all the basics of using SkiaSharp - you can find all that out in the old Xamarin.Forms [docs](https://learn.microsoft.com/en-us/previous-versions/xamarin/xamarin-forms/user-interface/graphics/skiasharp/). What I am going to discuss here is how to use SkiaSharp with MVVM in a .NET MAUI app.

Using SkiaSharp in .NET MAUI usually involves adding an `SKCanvasView` to your UI, and then handling it's `PaintSurface` event to draw graphical objects or images. The traditional way of doing this is by adding an `SKCanvasView` to a XAML page:

```xml
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:skiasharp="clr-namespace:SkiaSharp.Views.Maui.Controls;assembly=SkiaSharp.Views.Maui.Controls"
             x:Class="Imaging.Views.MainPage">            
    <skia:SKCanvasView PaintSurface="OnCanvasViewPaintSurface" />
</ContentPage>
```

Then, in the code-behind you have an event handler for the `PaintSurface` event:

```csharp
SKBitmap bitmap;

void OnCanvasViewPaintSurface(object sender, SKPaintSurfaceEventArgs args)
{
    SKImageInfo info = args.Info;
    SKCanvas canvas = surface.Canvas;

    canvas.Clear();
    canvas.DrawBitmap(bitmap, info.Rect, BitmapStretch.Uniform);
}
```

In the example above the `OnCanvasViewPaintSurface` event handler draws an `SKBitmap` on the `SKCanvasView`.

A common question I've seen asked is how to move the code that draws on the `SKCanvasView` (the `PaintSurface` event handler) from the view to a viewmodel? There are different approaches that could be used, but my preferred approach is to sub-class the `SKCanvasView` control and add a `BindableProperty` to it that specifies a service that's called to draw on the canvas. This service then needs implementing, injecting into a view model, and a method providing in the service that can be called from the view model to force the canvas to be re-drawn. While this approach adds considerable glue code, compared to the code-behind approach, the advantage is that it moves the code that draws on an `SKCanvasView` from the view to a service consumed by a viewmodel. This enables you to keep the code-behind for views largely empty, if that's what you desire.

## Sub-class the SKCanvasView

The first step is to sub-class the `SKCanvasView` and add a `BindableProperty` to it that specifies a service that's called to draw on the canvas:

```csharp
using SkiaSharp.Views.Maui.Controls;
using SkiaSharp.Views.Maui;
using Imaging.Services;

namespace Imaging.Controls;

public class MySKCanvasView : SKCanvasView
{
    public static readonly BindableProperty CanvasRendererProperty =
        BindableProperty.Create(nameof(CanvasRenderer), typeof(IBitmapRendererService), typeof(MySKCanvasView), null, defaultBindingMode: BindingMode.TwoWay,
        propertyChanged: (bindable, oldValue, newValue) =>
        {
            ((MySKCanvasView)bindable).CanvasRendererChanged((IBitmapRendererService)oldValue, (IBitmapRendererService)newValue);
        });

    public IBitmapRendererService CanvasRenderer
    {
        get { return (IBitmapRendererService)GetValue(CanvasRendererProperty); }
        set { SetValue(CanvasRendererProperty, value); }
    }

    void CanvasRendererChanged(IBitmapRendererService currentRenderer, IBitmapRendererService newRenderer)
    {
        if (currentRenderer != newRenderer)
        {
            if (currentRenderer != null)
                currentRenderer.InvalidateSurfaceRequest -= CanvasRendererInvalidateSurfaceRequest;

            if (newRenderer != null)
                newRenderer.InvalidateSurfaceRequest += CanvasRendererInvalidateSurfaceRequest;

            InvalidateSurface();
        }
    }

    void CanvasRendererInvalidateSurfaceRequest(object sender, EventArgs e)
    {
        InvalidateSurface();
    }

    protected override void OnPaintSurface(SKPaintSurfaceEventArgs e)
    {
        CanvasRenderer.PaintSurface(e.Surface, e.Info);
    }
}
```

In this example, the `BindableProperty` is called `CanvasRenderer` and is of type `IBitmapRendererService` which is a interface that specifies the operations the service will provide. When the `CanvasRenderer` bindable property is set to an `IBitmapRendererService` implementation, an event handler for the `InvalidateSurfaceRequest` event is registered that simply calls the `SKCanvasView` `InvalidateSurface` method to force the canvas to be re-drawn. The sub-classed control also overrides the `OnPaintSurface` method, which in turn calls a `PaintSurface` method that's specified in the `IBitmapRendererService` interface.

## Define the service interface

The `IBitmapRendererService` interface simply defines the operations that will be supported by the service that implements it:

```csharp
using SkiaSharp;

namespace Imaging.Services;

public interface IBitmapRendererService
{
    SKBitmap Bitmap { get; set; }
    void PaintSurface(SKSurface surface, SKImageInfo info);
    void InvalidateSurface();
    event EventHandler InvalidateSurfaceRequest;
}
```

## Implement the service

The `BitmapRendererService` implements the `IBitmapRendererService` interface, providing the `PaintSurface` implementation:

```csharp
using SkiaSharp;
using Imaging.Extensions;
using CommunityToolkit.Mvvm.ComponentModel;

namespace Imaging.Services;

public class BitmapRendererService : ObservableObject, IBitmapRendererService
{
    SKBitmap _bitmap = null;
    public SKBitmap Bitmap
    {
        get => _bitmap;
        set
        {
            SetProperty(ref _bitmap, value);
            InvalidateSurfaceRequest?.Invoke(this, EventArgs.Empty);
        }
    }

    public void PaintSurface(SKSurface surface, SKImageInfo info)
    {
        SKCanvas canvas = surface.Canvas;
        canvas.Clear();

        if (_bitmap != null)
            canvas.DrawBitmap(_bitmap, info.Rect, ImageStretch.Uniform);
    }

    public void InvalidateSurface()
    {
        InvalidateSurfaceRequest?.Invoke(this, EventArgs.Empty);
    }

    public event EventHandler InvalidateSurfaceRequest;
}
```

The `PaintSurface` method is used here to draw an `SKBitmap` to the canvas, and uses similar code to the earlier code for the `PaintSurface` event handler.

## Inject the service into the viewmodel

Provided that the `BitmapRendererService` type is registered with the `MauiAppBuilder` object (see the [example](https://github.com/davidbritch/skiasharp-imaging/blob/main/src/Imaging/MauiProgram.cs)), it can automatically be injected into viewmodels:

```csharp
public partial class MainPageViewModel : ObservableObject
{
    readonly IBitmapRendererService _bitmapService;
    public IBitmapRendererService BitmapRenderer
    {
        get => _bitmapService;
    }

    public MainPageViewModel(IBitmapRendererService bitmapService)
    {
        _bitmapService = bitmapService;
    }
    ...
}
```

## Consume the sub-classed `SKCanvasView`

The sub-classed `SKCanvasView` object can then be consumed from a page:

```xml
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:controls="clr-namespace:Imaging.Controls"                
             xmlns:viewmodels="clr-namespace:Imaging.ViewModels"
             x:Class="Imaging.Views.MainPage"
             x:DataType="viewmodels:MainPageViewModel">
    <controls:MySKCanvasView CanvasRenderer="{Binding BitmapRenderer}" />
</ContentPage>
```

In this example, the `CanvasRenderer` bindable property is set to the `BitmapRenderer` property from `MainPageViewModel`, with the `BindingContext` of the page being set to the viewmodel in code-behind.

Then, when the `Bitmap` property in the `BitmapRendererService` object is set to an `SKBitmap` it will be drawn to the sub-classed `SKCanvasView`. In addition, in the viewmodel whenever you want to force the sub-classed `SKCanvasView` to be re-drawn simply call the `InvalidateSurface` method on the `BitmapRendererService` object:

```csharp
_bitmapService.InvalidateSurface();
```

If you want to see this approach in action in a .NET MAUI app, go and check out [this repo](https://github.com/davidbritch/skiasharp-imaging). It contains a prototype that shows how to perform 2D image processing with SkiaSharp, hosted in a .NET MAUI app that targets Mac Catalyst.

If you want to see other examples of using SkiaSharp in .NET MAUI, check out the [SkiaSharp samples](https://learn.microsoft.com/samples/browse/?expanded=dotnet&products=dotnet-maui&terms=SkiaSharp) that I wrote while I was still at Microsoft.