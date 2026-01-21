---
title: Implementing F5 steganography in C#
date: 2026-01-21
excerpt: The F5 steganography algorithm shuffles the quantized DCT coefficients, and then uses matrix encoding to embed your secret message into the shuffled coefficients
tags: 
- "SkiaSharp"
- ".NET MAUI"
category: software
---

<a href="https://github.com/davidbritch/skiasharp-steg/tree/main/src/02%20-%20F5%20steganography" class="btn btn--info">Download the code</a>

[Previously]({% post_url 2025-11-11-implementing-jpeg-encoding %}) I discussed implementing JPEG compression in C#. This was as part of a steganography app for hiding information in JPEG images, which in turn necessitated implementing JPEG compression myself.

F5 is an algorithm for steganography that can be applied to several different image formats, including JPEG. In essence, the F5 algorithm shuffles the quantized DCT coefficients, and then uses matrix encoding to embed your secret message into the shuffled coefficients. If you want to know how it works in detail, you can read the [original paper](https://digitnet.github.io/assets/pdf/f5-a-steganographic-algorithm-high-capacity-despite-better-steganalysis.pdf) by Andreas Westfield. However, a high level overview of it as follows:

1. Begin the JPEG compression process but stop after the quantisation phase.
1. Initialise a cryptographically strong random number generator with a key derived from your password.
1. Instantiate a pseudo-random permutation of all available quantised DCT coefficients based on the key.
1. Embed your secret message across the shuffled DCT coefficients through matrix encoding.
1. Continue with the rest of the JPEG compression process (entropy encoding).

The advantages of this approach are that the total number of modifications to the image are reduced, making it harder to detect the hidden message via statistical analysis, and that the modifications are spread uniformly across the image.

I had intended writing multiple detailed blog posts that explain how the F5 steganography algorithm works, and how to implement it in a C#-based Mac app. But other interests have got in the way, and my enthusiasm for explaining all this has vanished. If you're interested, you should be able to figure it out from the [code](https://github.com/davidbritch/skiasharp-steg/tree/main/src/02%20-%20F5%20steganography).
