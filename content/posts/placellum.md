---
title: "Mini r/place!"
date: 2026-01-07T22:00:00Z
BookToc: false
Summary: "My first Node.js app! I love backend development!"
draft: false
---

{{% hint info %}}
**TLDR**

The idea behind polyslew is that at each sample I compute the numerical derivatives (first, second and third) using the most recent samples (2, 3 and 4 respectively). I then alter the most recent sample such that the numerical derivative does not exceed a specified bound. The math below gives the formula to calculate the most recent sample given the numerical derivative. In practice, I usually do the following at each sample: calculate the third derivative, set the sample, calculate the second derivative, set the sample, ... limit the signal, and shift the samples back for the next step.
{{% /hint %}}

