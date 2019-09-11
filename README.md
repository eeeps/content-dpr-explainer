# Content-DPR explainer

By default, browsers display images at their [“density-corrected intrinsic size”](https://html.spec.whatwg.org/multipage/images.html#density-corrected-intrinsic-width-and-height). [An 800×600, 1x image will display at 800×600 CSS `px`](https://codepen.io/eeeps/pen/mdbmbPq). [A 1600×1200, 2x image will *also* display at 800×600 CSS `px`](https://codepen.io/eeeps/pen/mdbmbEq). Even though the two images have different resource dimensions, as far as layout is concerned, they are identically-sized.

[The `Content-DPR` response header](https://whatpr.org/html/3774/e32a6f8...ddb0544/images.html#content-dpr) gives servers control over the density part of that equation, allowing them to serve arbitrarily-scaled responses without affecting (and breaking) layouts.

## Example

Let’s say a server sends `400x300.jpg` with the following header:

```
Content-DPR: 0.5
```

This instructs the user agent to give the image an intrinsic density of 0.5x. Thus, the image will have a density-corrected intrinsic size of 800×600 CSS `px`, no matter what markup or CSS delivers it.

## Why?

`Content-DPR` was invented in conjunction with [the `DPR` `Viewport-Width`, and `Width` client hints](https://whatpr.org/html/3774/e32a6f8...ddb0544/images.html#image-related-client-hints-request-headers), so that servers could respond to these hints without breaking layouts, performing [device-pixel-ratio-based selection](http://usecases.responsiveimages.org/#device-pixel-ratio-based-selection), [viewport-size-based selection](http://usecases.responsiveimages.org/#viewport-based-selection), and [resolution-based selection](http://usecases.responsiveimages.org/#resolution-based-selection) on the server-side.

Notably, the header solves other important use cases, all on its own.

But first—

## The client hints use case 

Let’s say a page author has chosen to craft a variable-device-pixel-ratio responsive image, with `srcset`.

Their markup might look like this:

```html
<img
	src="small.jpg"
	srcset="medium.jpg 2x,
	        large.jpg  3x"
	alt="A DPR-responsive JPEG"
/>
```

No matter which resource the browser selects, the density-corrected intrinsic size of the `<img>` will always be the same.

Here’s the equivalent client hints markup:

```html
<img
	src="client-hints.jpg"
	alt="A DPR-responsive JPEG"
/>
```

Let’s say this request goes out from [a 2.6x device](https://vizdevices.yesviz.com/devices/google-pixel2/), with the following hint:

```
DPR: 2.6
```

But the server only has 1x, 2x, and 3x versions readily available. Reasonably, it responds with the 2x version, along with the following header:

```
Content-DPR: 2
```

This ensures that, when the image arrives, the client knows its density, and it is given proper, consistent, density-corrected intrinsic dimensions.

## Other use cases

But even in browsers that don’t implement the `Width`, `DPR`, and `Viewport-Width` request hints, support for the `Content-DPR` response header would open up new opportunities for servers (and CDNs) to perform image optimization via resizing, without risking breaking layouts that rely on images’ default (intrinsic) size.

For example:

### Capping useful resolution

If a server receives a request for a very-large image, from a user agent that it knows does not have enough device pixels to take advantage of all of that resolution, it might reasonably choose to send a smaller version of the image. Currently, there is no way to do that without affecting the image’s intrinsic dimensions, potentially breaking layouts. `Content-DPR` would solve this.

### Prioritizing speed over quality

If the server has been given some signal that the user is in a constrained-bandwidth context (perhaps [an explicit signal, like `Save-Data: on`](https://wicg.github.io/netinfo/#save-data-request-header-field), or, perhaps an inferred one, based on measurements of previous response deliveries), it might want to respond with a down-scaled, low-resolution image. Again, this is impossible without either risking breaking some layouts, or universal support for `Content-DPR`.

### Low-quality image placeholders

Authors [commonly](https://jmperezperez.com/more-progressive-image-loading/) serve up [low-quality versions of images as placeholders](https://www.guypo.com/introducing-lqip-low-quality-image-placeholders), so that *something* is visible immediately, even if the full image load has been intentionally deferred (via lazy-loading), or is simply slow.

Right now, these techniques require some work on the front end to ensure that the downscaled placeholder is stretched to match the final dimensions of the full image. `Content-DPR` would allow servers to implement foolproof features that could respond to requests, for, say,

```
https://give.me/the-img.jpg?placeholder=true
```

with downscaled images that behaved just like their full-scale counterparts, as far as layout is concerned.

---

## Glossary


<dl>
<dt>Replaced element</dt>
<dd>

Replaced elements are external things that are embedded in an HTML document. Common replaced elements include `<img>`, `<video>` `<iframe>`, `<object>`,  and `<embed>`.

[MDN article on “replaced elements”](https://developer.mozilla.org/en-US/docs/Web/CSS/Replaced_element)

</dd>

<dt>Intrinsic dimensions</dt>
<dd>

Like all replaced elements, `<img>` elements have “intrinsic dimensions.” Which is to say: they have their *own width and height.* `<img>` elements' final, laid-out display dimensions can be *modified* by CSS, but, left to their own devices, they'll occupy a fixed width and height on a webpage, based on the dimensions of the embedded image resource.

For example: if you save a JPEG at a resolution of 800×600, when you stick it in an `<img>` element, by default, that `<img>` will have an intrinsic width of 800 CSS `px` and an intrinsic height of 600 CSS `px`.

[Spec definition of “intrinsic dimensions”](https://drafts.csswg.org/css2/conform.html#intrinsic).

</dd>

<dt>Device (physical) pixel</dt>
<dd>

One pixel on a physical display.

[Here's a zoomed-in picture of a bunch of device pixels](https://en.wikipedia.org/wiki/Pixel#/media/File:Closeup_of_pixels.JPG).

</dd>

<dt>CSS pixel</dt>
<dd>

One `px` in CSS.

Because the creators of CSS were thinking ahead, the size of `1 px` is not strictly tied to the size of device pixels – or even to any kind of fixed, measurable length. Instead, [the length of a `px` is loosely tied to an *ideal visual angle.*](http://inamidst.com/stuff/notes/csspx) 🤯

However, (probably because of its name?) for many years, web developers generally assumed that`1 px` == the length of one device pixel. This was a mostly-true (if fundamentally flawed) assumption until ~2012, when the advent of high-density (aka Hi-DPI, or ”Retina“) displays (containing very small pixels, very tightly packed) broke this assumption for good.

[Spec definition of the `px` unit](https://drafts.csswg.org/css-values/#px)

</dd>

<dt>Device Pixel Ratio</dt>
<dd>

A ratio, of (linear) *physical device pixels* to *CSS pixels*. Every browsing context has one, set by the device/browser, and accessible to developers via `window.devicePixelRatio`.

Before the advent of Hi-DPI displays, this ratio was usually 1:1 (aka, 1). Now, it can be all kinds of things.

For example: on my iPhone, Mobile Safari is using a device pixel ratio of 2:1 (aka 2). This means that when I make an element `1px` wide, when I view it on my phone, it will stretch linearly across two physical device pixels.

[dpi.lv](http://dpi.lv) is a fantastic resource which lists many different devices' resolutions and device pixel ratios.

</dd>

<dt>Image (logical) pixel</dt>
<dd>

One pixel in a digital image. All [bitmap images(https://en.wikipedia.org/wiki/Raster_graphics) are rectangular grids of logical pixels.

For instance, a JPEG with a resolution of 800×600 is 800 logical pixels tall, and 600 logical pixels wide.

</dd>

<dt>Image density</dt>
<dd>

A ratio, of (linear) *logical pixels* to *CSS pixels*.

For instance, if I give an image resource an image density of 2:1 (aka 2x), I am telling the browser to, by default, paint two linear logical pixels within every linear CSS pixel.

If the image density of an image resource and the device pixel ratio of a browsing context are the same, then, by default, every logical pixel will be painted cleanly within exactly one physical device pixel.

In HTML, developers can set this density via `srcset`'s `x` descriptors. e.g.

```html
<img srcset="hi-res.jpg 2x, low-res.jpg 1x" />
```

[Here's the spec definition of an `<img>` element’s “current pixel density”](https://html.spec.whatwg.org/multipage/images.html#current-pixel-density), which may be more confusing than helpful, honestly.

</dd>

<dt>Density-corrected intrinsic size</dt>
<dd>

The intrinsic size of an image, in CSS pixels, after taking into account both

1. its logical dimensions, and
2. its image density

For instance, if I have an 800×600 JPEG, but give it an image density of 2x, it will have a density-corrected intrinsic size of 400×300.

This size is available to developers via the `.naturalWidth` and `.naturalHeight` properties of `<img>` elements, in the DOM.

[Spec definition of density-corrected intrinsic dimensions](https://html.spec.whatwg.org/multipage/images.html#density-corrected-intrinsic-width-and-height)

</dd>
