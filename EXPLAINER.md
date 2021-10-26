Status: This explainer has been deprecated in favor of the [Segments Property Explainer](https://github.com/WICG/visual-viewport/blob/gh-pages/segments-explainer/SEGMENTS-EXPLAINER.md) under the Visual Viewport API spec.

# Window Segments Enumeration API

[Daniel Libby](https://github.com/dlibby-), [Zouhir Chahoud](https://github.com/Zouhir)

## Related explainers:
| Name | Link |
|------|------|
| Screen Fold API | [Explainer](https://github.com/SamsungInternet/Explainers/blob/master/Foldables/FoldState.md), [Unofficial Draft Spec](https://w3c.github.io/screen-fold/) |
| Multi-Screen Window Placement API | [Explainer](https://github.com/webscreens/window-placement/blob/master/EXPLAINER.md), [Draft Community Group Report](https://webscreens.github.io/window-placement/) |
| Visual Viewport API | [Draft Community Group Report](https://wicg.github.io/visual-viewport/), [MDN](https://developer.mozilla.org/en-US/docs/Web/API/Visual_Viewport_API) |

## Motivation:
Web developers targeting foldable devices want to be able to effectively lay out the content in a window that spans multiple displays. However, the web platform does not yet provide the necessary primitives for building layouts that are optimized for foldable experiences.
Developers may be able to solve this by taking a hard dependency on a specific device hardware parameters - an approach that is fragile, not scalable, and requires work duplication for each new device.

### Current problems:
More specific challenges we've heard from our internal product teams that were exploring building experiences for this emerging classes of devices include:

- *Hardware differences*: Devices could be seamless (e.g. Samsung Galaxy Fold) or have a seam (e.g. [Microsoft Surface Neo](https://www.microsoft.com/en-us/surface/devices/surface-neo), [Microsoft Surface Duo](https://www.microsoft.com/en-us/surface/devices/surface-duo) or ZTE Axon M). In the latter case developers might want to take it into account or intentionally ignore depending on scenario;
- *Folding capabilities, state*: the fold area could be safe or unsafe region to present content;
- *Future-proofing*: Ideally developers would want a somewhat stable way to target this class of devices without having to rely on specific device hardware parameters.

### Complementary existing proposals:
Before discussing the solution proposal - let's overview existing proposals that are relevant and applicable to the problem space. 
As matter of principle we should generally avoid creating redundant concepts if we can reuse existing platform APIs and capabilities.

- [Presentation API](https://w3c.github.io/presentation-api/) is solving the problem of a effective use of a _secondary_ screen and likely won't attempt to solve challenges outlined above that are specific to devices where a window can span separate physical displays. This would likely still be a separate problem for foldables

- [Screen Enumeration API Explainer](https://github.com/webscreens/screen-enumeration/blob/master/EXPLAINER.md) provides information about the physical screen configuration. Web developers might be able to leverage that on foldables, but would need to do extra effort to correlate that information with window parameters. Some concrete examples on why a special purpose API might be useful in addition to Screen Enumeration:
	- Getting adjacency information about spanning window regions to lay out content in several areas in logical way for a device;
	- Getting inner window dimensions that account for application frame, OS UI elements, etc.
- [Window Placement API Explainer](https://github.com/webscreens/window-placement/blob/master/EXPLAINER.md) is useful in multi-window scenarios on multiple screen devices, but does not target scenarios in which the hosting application (i.e. browser) has a single window which spans multiple displays. In this case, the developer may not wish to open new windows - just hints to help lay out things properly and take advantage of the physical partitioning of the available layout space.
 
Additionally, while not a solution in the same sense, a ["[css-media-queries] Foldables support and enablement"](https://github.com/w3c/csswg-drafts/issues/4141) issue discusses the problem space and outlines some details and touches upon outlined issues.

## Proposal: Window Segments Enumeration JavaScript API 

A summary of the concepts from the other proposals:
* Display - the logical representation of an physical monitor.
* Screen - the aggregate 2D space occupied by all the connected displays.

We propose a new concept of Window Segments that represent the regions (and their dimensions) of the window that reside on separate (adjacent) displays. Window Segment dimensions are expressed in CSS pixels and will be exposed via a JavaScript API that allows developers to enumerate segments where logically separate pieces of content can be placed. 

This proposal is primarily aimed at reactive scenarios, where an application wants to take advantage of the fact that it spans multiple displays, by virtue of the user/window manager placing it in that state. It is not designed for scenarios of proactively placing content in a separate top-level browsing context on the various displays available (this would fall under the [Window Placement API](https://github.com/webscreens/window-placement/blob/master/EXPLAINER.md) or [Presentation API](https://w3c.github.io/presentation-api/)). Note that given the [Screen Enumeration API](https://github.com/webscreens/screen-enumeration/blob/master/EXPLAINER.md) and existing primitives on the Web, it is possible to write JavaScript code that intersects the rectangles of the Display and window, while taking into account devicePixelRatio in order to compute the interesting layout regions of a window spanned across displays. However this may not correctly handle corner cases of future device form factors, and thus this proposal tries to centralize access to "here are the interesting parts of the screen a developer can target or consider for presenting content" as a practical starting point.

```
partial interface Window {
	sequence<DOMRect> getWindowSegments();
}
```

The value returned from the `getWindowSegments()` API will be an array of DOMRects, based on the data returned for each "window segment", developers will be able to infer the number of hinges available as well as the hinge orientation.

A user may at any point take the browser window out of spanning mode and place it on one of the screens or vice-versa, in those cases the window resize event will fire and authors can query and get the number of available screen segments.

This proposal doesn't aim to substitute existing APIs &mdash; the proposed development model can be summarized as requesting current window segments on interesting events and adjusting to the new presentation environment. There are no additional lifecycle proposals - the window segments are immutable and developers would request them upon common sense events (e.g. orientationchange, resize). It also  doesn't suggest how developers would use window segments to position, scale and orient content - in practical explorations developers used window segments to select the best declarative layout, not to modify layouts in script, but either would be possible.

## Security and Privacy

### APIs avalibility in iframe context

This API will be available in `iframe` context but disabled by default for privacy and security considerations. An author may enable them using the `screen-spanning` policy; a new feature policy we are proposing that will enable authors to selectively enable the JavaScript construct in iframe context. When disabled, getWindowSegments will return a single segment the size of the iframe's viewport.

iframes where `screen-spanning` feature policy is enabled will receive values in the client coordinates of the top most window, and it's possible the iframe won't be able to interpret that data without other information from its embedder. As an example, for cross origin iframes, the iframe's embedder must provide information about how to transform from the root client coordinate space to the iframe's client coordinate space, as this information is not available to cross-origin iframes for security reasons. 

## Examples of user experiences and solution outlines that can leverage two screens:

Let's take a look at a few practical examples of the scenarios above and how window segments would allow to resolve them for better user experience. In each case we'll start with some existing scenario and complicate it to provide opportunity to apply the proposal.

### A map application that presents a map on one window segment and search results on another

![Foldable with the left segment of the window containing a map and the right segment containing list of search results](map-app.svg)

#### JavaScript solution outline:

```js  
const screenSegments = window.getWindowSegments();

if( screenSegments.length > 1 ) {
	// now we know the device is a foldable
	// it's recommended to test whether screenSegments[0].width === screenSegments[1].width
	// and we can update CSS classes in our layout as appropriate 
	document.body.classList.add('is-foldable');
	document.querySelector('.map').classList.add('flex-one-half');
	document.querySelector('.locations-list').classList.add('flex-one-half');
}
```

### Reacting to map application resize/spanning state change

![Foldable with the left segment of the window containing browser and location finder website, right segment containing calculator app](map-app-resized.svg)

#### JavaScript solution outline:

```js  
window.onresize = function() {
	const segments = window.getWindowSegments();
	console.log(segments.length) // 1
}
```
