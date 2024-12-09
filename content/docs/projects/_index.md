---
bookCollapseSection: false
weight: 20
title: Projects
---

Aside from my research, the following is a taste of some of my interests inside and outside school.

## Effective Reproduction Value (R<sub>t</sub>) of COVID-19 in the WKU Community

R<sub>t</sub> is a key measure of how fast a virus is growing. It’s the average number of people who become infected from an infectious person. If R<sub>t</sub> is above 1.0, the virus will spread quickly. When R<sub>t</sub> is below 1.0, the virus will stop spreading. These values were calculated on January 12, 2021 using data provided by WKU's Engineering & Applied Sciences Department. The number of reported positive case "counts include anyone who self-identifies as a member of the WKU community to contact tracers or the WKU COVID Response Team regardless of whether the individual lives on campus or has physically been on WKU's campus."-WKU COVID Dashboard

This project took the home page of my personal website when WKU was still reporting positive cases.

![WKU Rt](/WKU_Rt.jpg)

The model used to calcuate R<sub>t</sub> is the exact same model used in https://rt.live. All credits go to Kevin Systrom and Thomas Vladeck for their contributions. A GitHub repository for the code used in this model can be found [here](https://github.com/rtcovidlive/covid-model).

I am neither an expert in epidemiology nor statistics. I have not consulted experts in these fields. I am just a student interested in applying this model to my own campus. The following are known issues:
 - The model needs what is basically the positivity rate which changes over time as the group of people tested changes. The model may underestimate current values of R<sub>t</sub> because far more people without symptoms are tested, thus driving the positive percentage down and overstating a downward trend in cases.
 - The model relies on a distribution for the delay between infection and a reported positive test. Most of the data used to generate this distribution is from Germany early in the pandemic – it is possible this delay distribution looks different in the US and across variants.

Everyone has the right to freely accessible scientific knowledge, especially regarding the spread of COVID-19 in their community. My page was created to inform campus administration and those interested.


## FastGAN-pytorch

Made a few updates to FastGAN-pytorch to make [NathanGAN](https://github.com/NathanGillispie/FastGAN-pytorch). Results are eerie. I ended up using the model trained on the FFHQ dataset for a video. 

![GAN1](/GAN1.webp)
![GAN2](/GAN2.webp)
![GAN3](/GAN3.webp)
![GAN4](/GAN4.webp)

## VSTs

Digital signal processing is cool and I like make programs to do it for me sometimes. Nothing bad will happen if you load these in your DAW.

{{< button href="https://github.com/NathanGillispie/KiosPlugins" >}}GitHub: NathanGillispie/KiosPlugins{{< /button >}}

PolySlew is a new take on the slew limiter. If you look at a given sample under a taylor expansion (it's a discrete signal but assume it isn't), the constant term is clamped to a range of values with a traditional limiter. The slew limiter clamps the linear term, or the first derivative of the signal. PolySlew generalizes this to the cubic term and adds different algorithms. This plugin was made to be as transparent as possible. It will blow up your speakers. Please add a limiter and high-pass after it.

In the future I want to create similar plugins to iZotope RX 10. There's a function that minimizes peak signal by locally altering the phase of the frequencies. It took me a while to figure out how it works. Essentially, a windowing function splits the incoming signal into buffers and a bunch of random FFT phases are tried in parallel to minimize the peak signal. It's incredibly fast and efficient and results in no audible artifacts, although you should expect transient smearing. I hope to make a real-time VST version of this, although it would be non-deterministic unless I used a pseudo-random generator.

## Oculus Knot Visualizer

What a fun program.

{{< button href="https://github.com/NathanGillispie/oculus_knot_visualizer" >}}GitHub: NathanGillispie/oculus_knot_visualizer{{< /button >}}

![Knot Project](/oculus.webp)

## Other machine learning endeavors

Some processing code to visualize a model learning on the perceptron model. I eventually made it to backpropogation of the multilayer perceptron model but that's ugly C# code with no cool visuals.

![Perceptron](/perceptron.webp)