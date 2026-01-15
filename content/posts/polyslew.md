---
title: "Polyslew: the polynomial slew limiter plug-in"
date: 2025-11-22T18:00:00Z
params:
  math: true
BookToc: false
Summary: "Plugin the generalizes slew limiters to polynomials. I made this three years ago but I'm finally writing about it."
draft: false
---

You can download Polyslew (VST) and view the source on my [Github](https://github.com/NathanGillispie/KiosPLugins/releases/tag/VST).
I wrote the plugin three years ago, but I've found it very useful and decided to write
a bit about it.

{{< figure src="/polyslew.webp" width=60% >}}

Audio sample without polyslew, polyslew only, and together.

<div class="center">
<audio controls preload="auto">
    <source src="/polyslew-AB.mp3">
</audio>
</div>

***

I've always been fascinated by the way people can add effects to sounds, in movies,
games, and music. I grew up making music digitally, mostly starting in Audacity, then
moving to Ableton in 2018. It was then when I learned about plug-ins: these effects
packaged up into little applications you could often download on the internet. It's
still amazing that so many applications for making music (DAWs) support standards
that allow you to make these plug-ins. It turns out, the process for writing these
plug-ins isn't special. Some developers may have a license with Steinberg that allows
them to make money by selling plug-ins with Steinberg's VST. But at the end of the
day, you're writing code that takes in a stream of bits, does something, then returns
a stream of bits. For as long as I knew about plug-ins, I wanted to make one, but
that seemed out of reach until one person.

It was actually a friend that told me about this open-source plug-in developer, Chris
from [AirWindows](https://www.airwindows.com/), which I highly recommend checking
out. He cranks out these plug-ins at least once a week. As far as I know, he's not
doing a challenge or anything, he's just autistic[^1] and probably retired. Actually,
there's another important reason he's able to make so many plugins. He basically never
creates a graphical interface. This has all the benefits you would expect: saving
massive amounts of time not fine-tuning something you can never hear in the final
mix. DAWs that support VST must have a way of controlling parameters of the plug-in.
This is typically for MIDI support, but that means most DAWs have sliders that you
can manually change. This feature, the default slider, is our saving grace. Personally
though, I completely agree with all his reasons for not writing GUIs, but I have one
extra reason: I don't know how to make them.

So I find out about AirWindows, I know a bit of C++, and I have a mission: make a
plug-in. The issue is what. I don't want to think about audio buffers (KISS), which
means I'm limited to writing code that processes one sample at a time. The idea I
came up with was one I had a while back, when I was interested in modular synthesizers.
Common in the modular scene is this thing called a slew limiter. It limits a signal's
first derivative in time. If you're not familiar with calculus, you might as well
stop reading and just use the plugin. The next step is obvious if you know calculus.
What about limiting the second-derivative of a signal? Third? FOURTH?? It's funny
how the language we use to describe things allows for certain ideas to naturally arise.
If you're used to thinking about slew limiters as "integrators" or "lag generators",
you probably wouldn't think there was anything special about fitting a parabola to
your signal and limiting the next extrapolated sample. But having a familiarity with
Taylor's theorem makes that idea a bit more reasonable.

That's enough foreshadowing. The plugin sucks, but in a really good way. In a way
that's hysterical (it literally has hysteresis), and allows for really creative uses.
When I first used used it, I thought it was broken, because things get really weird
when you do cubic limiting.

{{% hint info %}}
**TLDR on the math**

The idea behind polyslew is that at each sample I compute the numerical derivatives
(first, second and third) using the most recent samples (2, 3 and 4 respectively).
I then alter the most recent sample such that the numerical derivative does not exceed
a specified bound. The math below gives the formula to calculate the most recent sample
given the numerical derivative. In practice, I usually do the following at each sample:
calculate the third derivative, set the sample, calculate the second derivative, set
the sample, ... limit the signal, and shift the samples back for the next step.
{{% /hint %}}

## Refresher on numerical differentiation

Our signal \(f(x)\) is floating-point PCM, sampled at a constant interval \(h\), but
let's assume everything is real for simplicity. The taylor expansion of \(f(x)\) about
our current sample at \(x\) gives a formula for the value of the signal one sample
behind $$f(x-h)=f(x)-hf'(x)+\frac{h^2}{2!}f''(x)-\frac{h^3}{3!}f'''(x)+\cdots$$By
rearranging and dividing by \(h\),
$$\frac{f(x-h)-f(x)}{h}=-f'(x)+\frac{h}{2!}f''(x)+\cdots$$
You get exactly what you would expect for the derivative:
$$f'(x)=\frac{f(x)-f(x-h)}{h}+\mathcal{O}(h)$$
To get the formula for the second derivative, we need to consider another sample, now
two behind our current sample.
$$f(x-2h)=f(x)-2hf'(x)+\frac{4h^2}{2!}f''(x)-\frac{8h^3}{3!}f'''(x)+\cdots$$
The trick is to try and cancel terms by subtracting. The specific terms to subtract depends
on how many samples you're using and what approximation order you want. We want the
minimum (three-sample) second derivative formula, so let's cancel the linear terms.
$$f(x-2h)-2f(x-h)=-f(x)+h^2f''(x)-h^3f'''(x)+\cdots$$
Rearranging and dividing by \(h^2\) gives
$$\frac{f(x)-2f(x-h)+f(x-2h)}{h^2}=f''(x)-hf'''(x)+\cdots$$
which is \(f''(x)+\mathcal{O}(h)\) using big-O notation. In other words, neglecting
linear and higher-order terms, we get the classic formula for the three-point second
derivative, except from the last three samples in time. A similar \(\{1,-3,3,-1\}\)
formula exists for the third-derivative.

In general, you can solve a inverse matrix problem to find what amounts of which coefficient give you the derivative you want, to the order you want. Here is the Mathematica code I used:
```mathematica
hs = {-5,-4,-3,-2,-1}; (* f(x-hs*h) *)
(* {0,0,6,0,0} gets 3rd-deriv and removes 1/3! factor *)
solution = Inverse[(hs^#)&/@{1,2,3,4,5}].{0,0,6,0,0};
fits = Series[f[x+# h],{h,0,Length[hs]+1}]&/@hs;
solution.fits //FullSimplify
Print[solution]
```
This gives `-17*f[x]/4+(f^(3))[x]*h^3-15/8 (f^(6))[x]*h^6+O[h]^7`, the result of adding
all \(f(x-5h),\space f(x-4h),\dots\) with `solution` \(\{-7/4,41/4,-49/2,59/2,-71/4\}\).
The interpretation of these results is
$$\begin{align*}f^{(3)}(x)+\mathcal{O}&(h^3)=\frac{17}{4}f(x)-\frac{71}{4}f(x-h)+\frac{59}{2}f(x-2h)\\&-\frac{49}{2}f(x-3h)+\frac{41}{4}f(x-4h)-\frac{7}{4}f(x-5h)\end{align*}$$

## Linear case (slew limiting)

We take the current sample and compute the first derivative with
$$f'(x)=\frac{f(x)-f(x-h)}{h}$$
We check and see if \(f'(x)\) is inside or outside the range we set. We then clamp
the value to be within that range. Now we set \(f'(x)\mapsto g\) to be the limited
value and solve for the current sample \(f(x)\) given that limit: $$f(x)=hg + f(x-h)$$
This is precisely what is happening inside the code. Note that after computing this
new value, we shift everything over. \(f(x)\) becomes \(f(x-h)\) in the next call
to `process` and the last few samples always stay stored in RAM.

## Quadratic case

As above, solving for \(f(x)\) given a limited \(f''(x)\mapsto g\) gives
$$f(x)=h^2g+2f(x-h)-f(x-2h)$$
Something new happens here that really matters from an audio perspective. Given an
incoming signal limited between -1 and 1, applying a slew limiter can never generate
a signal outside the original range. That is not the case for quadratic limiting.
For example, let \(h=1\). Given two samples, \(f(x-2h)=0\) and \(f(x-h)=-1\), if the
next sample were \(f(x)=0\), the second-derivative would be \(2\). Limiting this to
\(f''(x)\mapsto 1/2\) gives \(f(x)=-3/2\), which is outside the original range. Therefore,
we must apply the constant limiter last if we want our signal to remain within a range.

## Technical challenges

You may want to apply slew limiting to the quadratic limiting, or vice versa, and
that will give you different results. Ultimately, I decided it was best to (zero-order)
limit the signal last, to prevent a signal outside the \([-1,1]\) range from blowing
up your speakers. This is especially useful given the fact that these signals really
like to blow up. 

{{% hint note %}}
Virtually all DAWs use floating-point numbers to represent signals. Unlike 16-bit
and 32-bit signed integer WAV files, the possible values for samples are not evenly
spaced, and go outside the range \([-1,1]\). For 64-bit doubles, the maximum representable
sample is about \(1.8\times 10^{308}\), which is about 3082dB larger than 1 (+3082dBFS).

However, I've noticed that Ableton places some type of compressor on signals that
are around +170dBFS, which is very strange and utterly useless. 
{{% /hint %}}

Also, you'll see no mention of \(h\) in my code. The reason for this is that I must
hard code limits into the parameters. If I set the max second derivative to be \(4\times44100\),
then the max second derivative only occurs when three consecutive samples read something
like \(\{1,-1,1\}\) at 44100Hz samling rate. My limiter will not have an effect on
the audio, as expected. If anyone changes the sample rate to be 48000Hz, this hard-coded
limit may not be enough to never have an effect on the audio. I could plan for the
worst and set this value super high, but my slider would miss out on valuable real
estate. On the other hand, I could set the limit to track with the sample rate, but
that's the same as just setting \(h=1\) and forgetting about sample rate, which is what I do.

## Cubic limiting is weird

It's easy for the signal to "blow up" when using cubic limiting. If we think of the
quadratic limiting as limiting the change in slope of the signal, cubic limiting would
limit the rate of change of concavity of the signal, without regards to the actual
slope. Therefore, cubic limiting alone can force a signal to be concave up or down
for a significant amount of time. Without limiting, the signal may become arbitrarily
large, even with normal inputs in \([-1,1]\) (I haven't proven this). With limiting,
the plugin can get stuck in an oscillatory state that requires careful manipulation
of the input samples to escape.

Given an input \(\{0,1,-1,0,0,0,\cdots\}\), applying cubic limiting (\(g\in [-1,1]\))
starting at the fourth sample causes the answer to diverge in an oscillatory manner,
visually similar to the graph of \(\cos(x)e^x\). If we limit the signal to \([-1,1]\)
after cubic limiting, we get an infinite oscillation of \(\{0,1,-1,-1,0,1,1,0,-1,-1,0,1,1,\cdots\}\).
You may have to change the settings or, in the worst case, turn off the plugin to
escape these oscillatory states. This is very impractical because the plugin may continue
playing horrid audio after pausing audio playback for as long as you let it.

I have not implemented the other numerical derivatives in the plugin, but they seem
to be more sensitive to diverging than the regular 4-point derivative. For example,
the 6-point derivative diverges given the trivial input \(\{0,.1,0,0,0,0,\cdots\}\)
and \(g\in [-1,1]\).

## Final thoughts

While writing this, I realized that there are multiple ways of limiting the derivative
aside from altering the most recent sample. I could alter any previous sample instead
of the most recent. I could also alter multiple samples at once. If we consider cubic
limiting, you can always alter the samples \(f(x-h)\) and \(f(x-2h)\) in such a way
that the signal never escapes the limiting bounds like the examples above. A similar
scheme exists for the second derivative: only altering the \(f(x-h)\) sample. The
issue of combining the two still persists. My intuition tells me there should be a
canonical way to generalize this scheme to higher derivatives (not altering the most
recent sample but multiple "middle" samples such that the signal never escapes \([-1,1]\)).
However, the problem of combining multiple types of limiting remains.

To solve this problem, I created a system of linear equations for the most recent
samples so that changing the third derivative (for example) keeps all other derivatives
the same. I haven't implemented this in plugin form, but tests seem promising.

Also, it would be interesting to see some sort of non-linear clipping instead of clamping
(hard clipping) the derivatives. That may be worth investigating later, but this is
all I have now!

[^1]: He describes himself as a "programmer with autism who's developing vanguard DSP..." on his website.

