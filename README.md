# Explainer: animation.overallProgress

## Introduction
This is an explainer for `animation.overallProgress`, a proposal to add an "overallProgress"
property to the JavaScript class
[Animation](https://developer.mozilla.org/en-US/docs/Web/API/Animation).

## Goal
The goal of this property is to provide authors a convenient and consistent
representation of how far along an animation has advanced across its iterations
and regardless of the nature of its
[timeline](https://developer.mozilla.org/en-US/docs/Web/API/AnimationTimeline).

### Background
There isn't a method or property which authors can use to directly know how much an animation has
advanced through its duration in a way that:
- accounts for all of its iterations,
- offers a consistent representation for both scroll-driven and time-driven
animations.

There does exist an [`AnimationEffect.getComputedTiming().progress`](https://developer.mozilla.org/en-US/docs/Web/API/AnimationEffect/getComputedTiming#progress) API but it only reflects the progress of the current iteration of the animation.

### Proposal
Add a read-only "overallProgress" field to [Animation](https://developer.mozilla.org/en-US/docs/Web/API/Animation)
which is represented the same way for both time-driven and scroll-driven
animations and accounts for the state of the animation and the actual
time/scroll range during which the animation is active.

## Use Cases
The proposed read-only `overallProgress` accessor allows authors to update other parts
of their page based on how far along an animation has advanced. They can use
this to:

- Synchronize another element's appearance, e.g. a video, according to the
animation's `overllProgress` as in the scroll-driven example below.
- Synchronize audio on a page with the progress of the animation as in the
time-driven example below.
- Give the user a sense of when an ongoing visual effect will end.


## Examples

### Time-Driven Animation

In this [demo](https://codepen.io/awogbemila/pen/oNKpXWy)
the [StereoPannerNode](https://developer.mozilla.org/en-US/docs/Web/API/StereoPannerNode)'s
[pan value](https://developer.mozilla.org/en-US/docs/Web/API/StereoPannerNode/pan#value)
is adjusted based on the progress of the time-driven animation moving the car
across the screen over several iterations. The audio is panned from left to right as the animation makes progress from the leftmost scene to the rightmost one. In this example, the author
starts an animation when the button is clicks and performs updates based on the
progress of the animation via [`requestAnimationFrame`](https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame) calls:
```
...
function progress() {
  return animation.overallProgress;
}

document.querySelector('#startbutton').addEventListener('click', (e) => {
  ...
  animation.play();
  const update = () => {
    adjustAudio(progress());
		...
  };

  requestAnimationFrame(update);
  startAudio();
});
...
```

### Scroll-Driven Animation

In this [demo](https://codepen.io/bramus/pen/BaXwmyZ)
the overall progress of a scroll-driven animation is used to set the video's current frame.

### Code Example

Here is a scroll-driven [demo](https://davmila.github.io/demo-animation.progress/sda/index.html)
where the developer synchronizes a textual indication of how far into a section of the text
the user has scrolled with a graphical indication (the green bar).

To do this, they create an animation using a scroll-timeline:
```
const makeAnimation = (subject, rangeStart, rangeEnd) => {
  const viewtimeline = new ViewTimeline(
    {
      subject: subject,
      axis: 'y'
    }
  );

  const animation = meterbar.animate(
  [
    { height: "0%" },
    { height: "100%" },
  ],
  {
    timeline: viewtimeline,
    rangeStart: rangeStart,
    rangeEnd: rangeEnd,
    fill: "forwards"
  });

  return animation;
}

const animation = makeAnimation(target, "cover 0%", "contain 100%");
```
and observe, during every scroll event, what the `overallProgress` of the animation is.

Since this animation is driven by scrolling, the developer could get the overall progress by
doing a few calculations based on [scrollTop](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollTop):
```
function progress() {
  const lower_bound = (target.offsetTop - scroller.offsetTop) - scroller.clientHeight;
  const upper_bound = lower_bound + target.offsetHeight;

  // Compute raw fraction.
  let progress = (scroller.scrollTop - lower_bound) / (upper_bound - lower_bound);

  // Clamp to [0, 1];
  progress = Math.min(progress, 1);
  progress = Math.max(progress, 0);

  return progress;
}
```
which is slightly more tedious than what they could do with `animation.overallProgress`:
```
function progress() {
  return animation.overallProgress;
}
```
They'd then update the textual indication on every scroll event.
```
const updateViewInfo = () => {
  ...
  const progress_pct = progress() * 100;
  viewInfo.innerHTML = `<h3>Required Section progress: ${ Math.round(progress_pct) }%.</h3>`;
  ...
};

scroller.addEventListener("scroll", updateViewInfo);
```
Note that the above `scrollTop` calculation is what corresponds to the
[rangeStart](https://developer.mozilla.org/en-US/docs/Web/API/Element/animate#rangestart)
and [rangeEnd](https://developer.mozilla.org/en-US/docs/Web/API/Element/animate#rangeend)
of "cover 0%" and "contain 100%" respectively
(in `makeAnimation()`) in this particular layout. It might need to
be adjusted slightly to correspond to a different scroll range and layout.

## Details

`Animation.overallProgress` is generally defined as
```
progress = currentTime / effect endTime
```
where `currentTime` is the [currentTime](https://developer.mozilla.org/en-US/docs/Web/API/Animation/currentTime)
of the animation and `effect endTime` is the [endTime](https://developer.mozilla.org/en-US/docs/Web/API/AnimationEffect/getComputedTiming#endtime) of the animation's [effect](https://developer.mozilla.org/en-US/docs/Web/API/Animation/effect).

It is however clamped to values in the range [0, 1] and will be null if:

- the animation's currentTime is null, or
- the animation has no effect.

If an animation's [endTime](https://developer.mozilla.org/en-US/docs/Web/API/AnimationEffect/getComputedTiming#endtime)
is zero, its `overallProgress` will be:

- 0 if its [currentTime](https://developer.mozilla.org/en-US/docs/Web/API/Animation/currentTime) is negative, and
- 1, otherwise.

If an animation has infinite  [endTime](https://developer.mozilla.org/en-US/docs/Web/API/AnimationEffect/getComputedTiming#endtime)
its progress is 0.

### Considerations & Trade-Offs

#### Clamping to [0, 1]

By clamping `animation.overallProgress` to [0,1] we handle the case of zero-duration 
animations by returning values of 0 or 1 which are more likely to be useful to
developers than the more mathematically correct `-Infinity` and `+Infinity`.
This however means that an animation `A` whose `currentTime` is 
less than its `startTime` and a non-zero-duration animation `B` whose
`currentTime` is equal to its `startTime` (and may therefore be visually 
reflecting the animation's having started) both report `overallProgress` of zero.
