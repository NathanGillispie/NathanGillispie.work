---
title: "Post-scarcity r/place"
date: 2026-01-07T22:00:00Z
BookToc: false
Summary: "I made a mini r/place using only Node and Express.js. First real backend development! You can play at minirplace.nathangillispie.work"
draft: false
---

During the first edition of [r/place](https://en.wikipedia.org/wiki/R/place), users started with a grid of 1000x1000 pixels. Anyone could edit them, but only once per 5-20 minutes. In the scarcity model, conflict emerges from constraint. Reddit manufactured scarcity in r/place by throttling interactions, creating the illusion that pixels were a finite natural resource rather than an arbitrarily rationed one. With only four pixels, the pretense of scarcity is paradoxically removed. Every action is simultaneously maximal and trivial. One user can overwrite the totality of the grid in 12 seconds.

<style>
.pixel {
    box-shadow: 0 0px 4px rgba(0, 0, 0, .4);
    border-radius: 4px;
    transition: background-color .14s linear, transform .07s ease-in;
    transform: scale(1);
}
@media (hover: hover) {
    .pixel:hover {
        transform: scale(.96);
    }
}
.pixel-active {
    transform: scale(.96);
}
</style>
<div id="pixel-grid" style="aspect-ratio: 1 / 1; width: 75%; max-width: 400px; margin: auto; display: grid; grid-template-columns: repeat(2, 1fr); grid-template-rows: repeat(2, 1fr); gap: 7px;">
    <div class="pixel"></div>
    <div class="pixel"></div>
    <div class="pixel"></div>
    <div class="pixel"></div>
</div>
<script>
const palette = ["#e6e4dc","#161717","#e63b15","#2039f5","#10eb4b","#11edba","#ef3dff","#f2da3a","#9a38eb"];
const getRandomColor = () => palette[Math.floor(Math.random()*9)];
const p = document.querySelectorAll(".pixel");
for (let i = 0; i < p.length; i++) {
    p[i].style.backgroundColor=getRandomColor();
    if ('ontouchstart' in window) {
        p[i].addEventListener("touchstart", ()=>{
            p[i].style.backgroundColor=getRandomColor();
            p[i].classList.add("pixel-active");
        });
        p[i].addEventListener("touchend", ()=>{
            p[i].classList.remove("pixel-active");
        });
    } else {
        p[i].addEventListener("pointerdown", ()=>{
            p[i].style.backgroundColor=getRandomColor();
            p[i].classList.add("pixel-active");
        });
        p[i].addEventListener("pointerup", ()=>{
            p[i].classList.remove("pixel-active");
        });
    }
}
console.log("This one couldn't wait until April 1 lol :)");
</script>

> "It is owning a body that makes us subject to calamities. If we possessed no body of our own, what calamity could we have?" 道德經, Chapter 13

[My version](https://minirplace.nathangillispie.work/) of r/place is nominally more limited, but it presents as a post-scarcity alternative showing the absurdity of limited resources in a world where technology has the capability of removing those limitations. This 2x2 canvas acknowledges that some limits are not technical, but ideological. Computers were designed for interoperability. The concept of private property is deeply embedded into our collective conscious, but any sufficiently knowledgable person knows how unnatural it is to, for example, ["loan" out](https://help.archive.org/help/borrowing-from-the-lending-library/) library books online.

> "Doors and windows are chiseled out to make a room---the function of the room happens where the absence is. So profit lies in the having of something, in properties possessed, but function lies in the having of nothing---in possession of no properties at all." 道德經, Chapter 11

By removing pixels, you reach a Hegelian *node*, the point where minute quantitative changes transform into qualitative ones. In our case, the space of expressable signifiers reduces such that 'drawing' becomes trivial. The concept of a portion of the board being 'yours', or part of a collective effort, vanishes. No longer are users burdened with deciding where to place their allegiance, or deciding whether to attack or defend. Everyone's work can neither be created nor destroyed.

> "Is not the space between Heaven and Earth like a bellows? Empty, but never exhausted. With every movement, more emerges." 道德經, Chapter 5

Though it may seem reductive, I'm presenting a more honest version of r/place. It does not pretend that territory can be conquered, or that meaning accumulates through scale. Instead, it offers erasure as the default state and invites users to participate anyway -- fully aware their contribution will be both noticed by everyone and immediately obsolete.

> "Better to stop short than to fill up all the way. A blade polished keen cannot be long preserved.
>
> To retreat once the work is done---that is the course of Heaven." 道德經, Chapter 9

{{< button href="https://minirplace.nathangillispie.work/" >}}Visit minirplace.nathangillispie.work{{< /button >}}

