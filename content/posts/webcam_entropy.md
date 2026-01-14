---
title: "Hardening Linux with AI-powered safety cameras by infusing `/dev/ranodom` with entropy"
date: 2026-01-13T16:00:00Z
BookToc: true
Summary: ""
---

Random numbers are required for generating long-term cryptographic keys. On linux, they can be found in the `/dev/random` file, which uses the linux kernel's cryptographically secure psuedo-random number generator (CSPRNG). Embedded machines and live CDs are particularly vulnerable to being hacked---having some outsider determine the exact state of the kernel's CSPRNG, which would compromise any keys generated. To seed the CSPRNG, the kernel gathers 'entropy' from various system sources like interrupt timings, hardware random number generators, or online sources like the [NIST randomness beacon](https://csrc.nist.gov/projects/interoperable-randomness-beacons/beacon-20).

Companies like SGI and Cloudflare have devised more creative sources of entropy. Infamously, Cloudflare uses a live video feed of lava lamps, called The Wall of Entropy. That reminds me... what if we used video feeds of equally random processes, like kids on a playground, or women walking alone outside. Thankfully, some AI-powered safety cameras have recently made headlines[^1] for allegedly providing us with this source of information!

The rest of this article will be about adding hypothetical video feeds as a source of entropy to your linux machine. Here's the general overview:
1. Video comes in through a `v4l2` device. I'm capturing a livestream with OBS and sending that out through its virtual camera, but you could just use a webcam input as well.
2. `ffmpeg` takes the time difference between each frame then combines the diffs by averaging pixels (scaling down with the area filter)
3. The output of `ffmpeg` is written to a FIFO.
4. A python script applies the inverse cumulative distribution function to obtain a uniform distribution, the output is written to another FIFO file.
5. `rngd` injects the bits into the entropy pool used by `/dev/random`
6. Rejoice! We are all more secure now that AI-powered safety cameras exist!
## How do the lava lamps work?
It's just an image of lava lamps put into a cryptographic hash function. The resulting value is used to seed a pseudo-random number generator. The specifics vary for SGI's original implementation[^2] (the OGs) vs Cloudflare's Wall of Entropy. There really is no reason for the video feed to be lava lamps. If you're using a cryptographic hash function, you just have to ensure every frame is unique, even if they differ by only one bit.

This works, but this isn't complicated enough. Besides, cryptographic hash functions are essentially black boxes. As far as I know, no proof exists for the distribution of the SHA-1 hash, as used in SGI's original entropy source. We are relying purely on emperical observation, hoping that the hash is cryptographically secure. I'm not here to bring SHA-1 into question of course, but I find it more pleasing if we can start with something with a known distribution, like noise on a camera sensor, and convert that to a uniform distribution using known math.
## Inverse transform sampling
My approach is the naive one, but if provides an excuse to learn some statistics. In essence, we use the central-limit theorem to obtain a normal distribution, then map that to the uniform distribution by the cumulative distribution function, which will be used to seed entropy for our pseudo-random number generator.

Inverse transform sampling is a way of sampling a random variable $X$ on $\mathbb{R}$ from some uniform $U$ on $[0,1]$. Our objective is to do this the other way around. This is possible for the normal distribution because its cumulative distribution is one-to-one.

Given a normal distribution with known population mean $\mu$ and variance $\sigma^2$, for our sample $x$, we map $F:x\mapsto [0,1]$ via the cumulative distribution function $F$: $$F(x)=\frac{1}{2}\left[1+\text{erf}\left(\frac{x-\mu}{\sigma\sqrt{2}}\right)\right]$$Now the challenge is to obtain samples of a normal distribution with known population mean and variance. We can approach this with the central-limit theorem.
### Central-limit theorem
One of the most fundamental ideas in probability theory is the central-limit theorem, which can be restated in a variety of ways. In short, if we take the mean of a sufficiently large number of samples from *any* distribution (with finite variance) we eventualy get the normal distribution. For our purposes, samples must be independent and identically distributed, but this requirement can be weakened.

> [!note] Galton board example
> The classic exmample given is that of the Galton board. In it balls falling down with pegs arranged in a triangular order. Assume there's a 50% chance that a ball will go to the left of the first peg. Then it will reach another peg, from which it will face a 50% chance of going to the left again. Suppose we take left as 0 and right as 1, then the sample is the average (or equivalently, sum) of all decisions made by a ball. Given a large enough number of rows, the central limit theorem states that the distributions of these samples (sums) will be approximately the normal ditribution. This is in fact reflected in the Galton board.

Let us assume the objects in our video are not moving relative to the camera frame, such that the differences frame to frame are dominated by camera noise. Let us also assume this noise is independent and identically ditributed. **Already these assumptions eliminate most outdoor cameras** since noise in digital sensors have a variety of sources that vary with the quantity of photons reaching the sensor, sensor temperature, exposure time, and so on.[^3]

The idea is that the difference between two frames in time is going to have *some distribution*, and if we take the average of this for enough pixels, we get the normal distribution by the central-limit theorem.
- The `tmix` video filter is used to take the difference of two adjacent frames in time.
- The averaging process can actually be done in FFmpeg by scaling down! To weight all pixels equally, we can use the `flags=area` option in the scale video filter.
First, we would benefit from some extra precision by changing the pixel format to a higher bit-depth.
### Pixel formats
YUV color models almost work but suffer from a similar issue RGB does. Negative values seem to be clamped to 0 immedately after the `tmix` filter within the filter chain. Our `tmix` filter maps a pixel channel to 0 when there are no differences in time (as expected), but when the chroma channels are 0, the whole image turns green. To combat this you could try to recenter the chroma channels (i.e. `lut='u=val+128:v=val+128'` for 8-bit channels), but half of colors are still inaccessible due to clamping.

This presents the first real technical challenge: **the signal must be in $[0,maxval]$ at all points in the filter chain.** You might think `rgb24`, etc. doesn't work because pixel values are made of unsigned integers. But FFmpeg supports IEEE 754 floating-point pixel formats, like `gbrpf32le`. Even these seem to be clamed to 0 after `tmix`. If I apply a lut to move pixel values in the range $[-\text{max}, \text{max}]$ to $[0, \text{max}]$ with `val*.5+maxval*.5`, the result makes zero map to middle grey `#888`, which is the darkest color in the output.
### My solution to clamped values from `tmix`
I think the best solution is to recreate the effect of `tmix` with a complex filter chain. If we take one signal, delay it by one frame, then invert it with `maxval-val`, both the input and output will remain in the desired range. Then we can combine the delayed+inverted signal to the same signal with the blend filter + average mode. Let's look at an example.

Here is a test video produced with `ffmpeg -f lavfi -i testsrc -t 6 -pix_fmt rgb24 gradient.mov`
==INSERT GIF==
And here is the output when I apply a one-second delay to the inverted signal before averaging the two chains together.
```
ffmpeg -i gradient.mov \
  -filter_complex "[0]tpad=start_duration=1:start_mode=clone,\
  lutrgb='r=maxval-val:g=maxval-val:b=maxval-val'[delayed];\
  [0][delayed]blend=all_mode=average[outv]" \
  -map "[outv]" output.mp4
```
==INSERT GIF==
Pixels that go from white to black appear as black. Pixels that go from black to white appear as white. Pixels that do not change appear as grey. 
### Wrapping up `ffmpeg`
Given a 30 fps framrate, our delay should be about 0.0333s.

Our script so far looks like this
```bash
ffmpeg -f v4l2 -i /dev/video0 \
  -filter_complex "[0]format=rgb48be[input];\
  [input]tpad=start_duration=0.0333:start_mode=clone,\
  lutrgb='r=maxval-val:g=maxval-val:b=maxval-val'[delayed];\
  [input][delayed]blend=all_mode=average,\
  scale=1:ih:flags=area[outv]"
  -f rawvideo -c:v rawvideo fifo_output
```
The output would be 16-bit for each channel. Assuming the differences in time are small, truncating to 8-bits would probably result in aliasing. This middle gray (no difference between pixels) is hex $\texttt{80 00}$. To make the output size small, we could bit shift left by at least 2-bits and get a valid two's compliment signed integer. If we bit shift left by 8-bits,$$\texttt{80 XX}\mapsto \texttt{XX}$$we get a signed int8. Let's look at some data from the same example above:
```
0047ce0  80  80  80  80  80  80  80  80  80  80  81  80  80  80  80  80
0047cf0  80  80  81  80  80  80  7f  7f  80  80  81  80  7e  7d  7d  7d
0047d00  7d  7c  7d  7c  7d  7d  7d  7c  7d  7c  7d  7c  7c  7d  7d  7c
0047d10  7d  7c  7d  7c  7d  7d  7d  7c  7d  7c  7d  7c  7d  7d  7d  7c
```
It appears 8 bits of bit shifting may be too much. Instead, we can perform bitwise operations with FFmpegs `geq` filter.
### `geq` bitshifting (to reduce file size)
> Honestly finding documentation on this filter was impossible, but check out [libavutil/eval.c](https://www.ffmpeg.org/doxygen/2.8/eval_8c_source.html) in the FFmpeg source code if you want to know more. `if` statements are implemented, and complex filters allow for loopback... I have a hunch that cellular automata are possible using just FFmpeg filters. Next project: "FFmpeg filters are turing complete"?

If we want to bitshift left by two bits, we need to mask out the first two bits, since values above max get clipped instead of just overflowing. We do this with bitwise `& 0x3fff`, followed by `*4`. In FFmpeg filter terms, we have
```
geq='r=bitand(r(X,Y),16383)*4:g=bitand(g(X,Y),16383)*4:b=bitand(b(X,Y),16383)*4'
```
inside the complex filters. Now we have
```bash
ffmpeg -f v4l2 -i /dev/video0 \
  -filter_complex "[0]format=rgb48be[input];\
  [input]tpad=start_duration=0.0333:start_mode=clone,\
  lutrgb='r=maxval-val:g=maxval-val:b=maxval-val'[delayed];\
  [input][delayed]blend=all_mode=average,\
  scale=1:ih:flags=area\
  geq='r=bitand(r(X,Y),16383)*4:g=bitand(g(X,Y),16383)*4:b=bitand(b(X,Y),16383)*4'\
  format=rgb24[outv]" \
  -f rawvideo -c:v rawvideo fifo_output
```


Once we have a normal distribution, we can apply the inverse cumulative distribution function


```bash

sudo rngd -f -r /tmp/vserial_out

```

---
[^1]: https://www.404media.co/flock-exposed-its-ai-powered-cameras-to-the-internet-we-tracked-ourselves/

[^2]: https://web.archive.org/web/19980521144845/https://lavarand.sgi.com/cgi-bin/how.cgi

[^3]: https://en.wikipedia.org/wiki/Image_noise#In_digital_cameras

