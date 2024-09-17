# Explainer: animation.progress

## Introduction
This is an explainer for `animation.progress`, a proposal to add a "progress"
property to the JavaScript class
[Animation](https://developer.mozilla.org/en-US/docs/Web/API/Animation).

## Goal
The goal of this property is to provide authors a convenient and consistent
representation of how far along an animation has advanced across its iterations
and regardless of the nature of its
[timeline](https://developer.mozilla.org/en-US/docs/Web/API/AnimationTimeline).

### Background
There isn't a mechanism by which authors can directly know how much an animation has
advanced through its duration in a way that:
- accounts for all of its iterations,
- accounts for whether its `currentTime` is before or after its `startTime`, and
- offers a consistent representation for both scroll-driven and time-driven
animations.

There does exist a [`getComputedTiming().progress`](https://developer.mozilla.org/en-US/docs/Web/API/AnimationEffect/getComputedTiming#progress) API but it only reflects the progress of the current iteration of the animation.

To compute the current progress of an animation, an author could use
its [currentTime](https://developer.mozilla.org/en-US/docs/Web/API/Animation/currentTime)
along with other properties of [Animation](https://developer.mozilla.org/en-US/docs/Web/API/Animation)
but they would need to account for `currentTime` not being represented the same
way for scroll-driven and time-driven animations (`currentTime` is a percentage
for scroll-driven animations and an absolute time quantity for time-driven animations).
Additionally, with scroll-driven animations, authors also have to account for
the fact that `currentTime` is measured relative to the whole scroll range
whereas a scroll-driven animation's [start](https://developer.mozilla.org/en-US/docs/Web/API/ViewTimeline/startOffset)
and [end](https://developer.mozilla.org/en-US/docs/Web/API/ViewTimeline/endOffset)
might not correspond to the entire scroll range.

### Proposal
Add a read-only "progress" field to [Animation](https://developer.mozilla.org/en-US/docs/Web/API/Animation)
which is represented the same way for both time-driven and scroll-driven
animations and accounts for the state of the animation and the actual
time/scroll range during which the animation is active.

## Use Cases
The proposed read-only `progress` accessor allows authors to update other parts
of their page based on how far along an animation has advanced. They can use
this to:

- Give the user a sense of when an ongoing visual effect will end, e.g. in the
time-driven flashing animation example below.
- Synchronize another element's appearance, e.g. a video, according to the
animation's `progress`.


## Examples

### Time-Driven Animation

In this [time-driven animation demo](https://davmila.github.io/demo-animation.progress/tda/index.html)
where the developer lets the user set the number of iterations of 
the flashing animation, the developer can provide the user 
feedback of the progress of the animation as time passes by
accessing `animation.progress`:

```
function animateBox() {
  ...

  animation = box.animate([
    { opacity: 1 },
    { opacity: 0 },
    { opacity: 1 }
  ], {
    duration: 1000,
    iterations: flashCount
  });

  ...
}

function updateInfo() {
  ...
  const progress_pct = Math.round(animation.progress * 100);
  info.innerHTML = `The animation's progress: <div>${progress_pct}%<div>`;
  ...
}
```

### Scroll-Driven Animation

In this [scroll-driven animation demo](https://davmila.github.io/demo-animation.progress/sda/index.html)
the developer has a convenient way of indicating to the user how far into a
particular section of the text on the page a user has scrolled.

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
and observe, during every scroll event, what the `progress` of the animation is:

```
const updateViewInfo = () => {
  ...
  const progress_pct = animation.progress * 100;
  viewInfo.innerHTML = `<h3>Required Section progress: ${ Math.round(progress_pct) }%.</h3>`;
  ...
};

scroller.addEventListener("scroll", updateViewInfo);
```

## Details

`Animation.progress` is generally defined as
```
progress = currentTime / effect endTime
```
where `currentTime` is the [currentTime](https://developer.mozilla.org/en-US/docs/Web/API/Animation/currentTime)
of the animation and `effect endTime` is the [endTime](https://developer.mozilla.org/en-US/docs/Web/API/AnimationEffect/getComputedTiming#endtime) of the animation's [effect](https://developer.mozilla.org/en-US/docs/Web/API/Animation/effect).

It is however clamped to values in the range [0, 1] and will be null if:

- the animation's currentTime is null, or
- the animation has no effect.

If an animation's [endTime](https://developer.mozilla.org/en-US/docs/Web/API/AnimationEffect/getComputedTiming#endtime)
is zero, its `progress` will be:

- 0 if its [currentTime](https://developer.mozilla.org/en-US/docs/Web/API/Animation/currentTime) is negative, and
- 1, otherwise.

If an animation has infinite  [endTime](https://developer.mozilla.org/en-US/docs/Web/API/AnimationEffect/getComputedTiming#endtime)
its progress is 0.

### Considerations & Trade-Offs

#### Clamping to [0, 1]

By clamping `animation.progress` to [0,1] we handle the case of zero-duration 
animations by returning values of 0 or 1 which are more likely to be useful to
developers than the more mathematically correct `-Infinity` and `+Infinity`.
This however means that an animation `A` whose `currentTime` is 
less than its `startTime` and a non-zero-duration animation `B` whose
`currentTime` is equal to its `startTime` (and may therefore be visually 
reflecting the animation's having started) both report `progress` of zero.
