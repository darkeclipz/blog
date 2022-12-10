---
title: "CSS tricks"
date: 2022-11-23T08:47:11+01:00
draft: true
---

<style>
    .box {
        display: flex;
        max-width: 36rem;
        margin: 0 auto;
        box-sizing: border-box;
        border: 1px black dashed;
    }

    .round {
        border-radius: 24px;
    }

    .center {
        justify-content: center;
    }

    .gap\:12 {
        gap: 12px;
    }

    .gap\:24 {
        gap: 24px;
    }

    .tag {
        padding: 6px 8px;
        background-color: #eee;
        border-radius: 4px;
        margin: 12px;
        text-transform: uppercase;
        font-family: sans-serif;
        font-size: 1rem;
    }

    .primary {
        background-color: #F3C677;


        background-color: #00000022;
    }  

    .secondary {
        background-color: #F9564F;
        background-color: #00000022;
    }

    .tertiary {
        background-color: #B33F62;
        background-color: #00000022;
    }

    .tag-fix .tag:first-child {
        margin-left: unset;
    }

    .tag-fix .tag:last-child {
        margin-right: unset;
    }
</style>

I came across this design pattern about a year ago, and it actually is only one of the few things that I actually use.
It is so powerful, that you can design almost any layout with it.
I have been working with CSS for well over a decade and the endless battles with padding and margin finally have been won.

The design pattern is coined with the name: **The Stack**, and if anyone has been using MUI or equivalents lately, you probably already know about it. A stack consists of a container, in which there is a set of child elements (which could also be containers).

<p>
    <div class="box gap:24">
        <div></div>
        <div></div>
        <div></div>
        <div></div>
        <div></div>
    </div>
</p>

## The stack

## Box-sizing: border-box

Anyone who has worked with CSS has probably ran into this issue at some point.
You are designing a container with a specific with and height, and then when you add a border, everything is screwed up.
Yeah, very annoying, and not sure why the default behaviour even is the way it is.
However, there is a fix to this problem.

The attribute `box-sizing` is set to `content-box` by default, which is defining the weird behaviour. 
There is the option `border-box`, which sizes the box the way you expect.
This snippet applies the `border-box` sizing to all of the elements in the document.

{{< highlight css >}}
* {
    border-sizing: border-box;
}
{{</ highlight >}}

Yay, no more weird sizing issues!

## Aspect-ratio

Suppose you have a few elements of which the size depends on the viewport.

<div class="box">

</div>

## Margin no more

In the day and age of flexbox, I am not really sure what purpose margin is left to fullfill.
More often then not, it adds complexity to the layouts and CSS rules.
This happens because _margin violates the encapsulation of components_ because it interacts with the space that is defined outside of the component.

Let's look into this with an example that gives a few issues we are probably all familiar with.
Suppose we have a bunch of labels, that have some space around it by applying `margin: 24px`.

{{< highlight css >}}
.tag {
    margin: 24px;
}
{{</ highlight >}}

<div class="box primary">
    <div class="tag secondary">Get</div>
    <div class="tag secondary">rid</div>
    <div class="tag secondary">of</div>
    <div class="tag secondary">margin</div>
</div>

The first issue is the extra margin that is in front of the first element, and at the back of the last element (although that is not visible). This is fixable by applying a bandage. In those cases we can remove the margin of the specific elements with `:first-child` and `:last-child`.

{{< highlight css >}}
.tag {
    margin: 24px;
}

.tag:first-child {
    margin-left: unset;
}

.tag:last-child {
    margin-right: unset;
}
{{</ highlight >}}

<div class="box tag-fix primary">
    <div class="tag secondary">Get</div>
    <div class="tag secondary">rid</div>
    <div class="tag secondary">of</div>
    <div class="tag secondary">margin</div>
</div>

However, _the encapsulation of the component is still broken_ because there is a margin on top and below the tags.
We can change the margin to be only defined on the left and the right, and be done with it.
This still won't solve all the issues though. 
What if we want to use the tag in another container, with a different margin?

To fix this problem we first need to remove any margin on the tag.
By doing so, the tag no longer violates the encapsulation principle, and is fully reusable.
Next we define the gap between the tags on the container level with the `gap` attribute.

{{< highlight css >}}
.container {
    display: flex;
    gap: 24px;
}
{{</ highlight >}}

<div class="box primary gap:24">
    <div class="tag secondary" style="margin: unset">Get</div>
    <div class="tag secondary" style="margin: unset">rid</div>
    <div class="tag secondary" style="margin: unset">of</div>
    <div class="tag secondary" style="margin: unset">margin</div>
</div>

> Reusability of components is maximized if components are encapsulated.

Instead of using margin, try positioning elements with `justify-content`, `align-items`, and if spacing is needed use `gap`. If this is done consequently, your layout will become a much simpler and maintainable.

