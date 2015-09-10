
# Why is async image conversion necessary?

Web apps doing image processing work may need to read or modify pixel data. This has traditionally been done through the CanvasRenderingContext2D `getImageData` and `putImageData` methods. This includes use cases where you simply want to convert between types, such as getting from `Blob` to `ImageData` which requires use of an intermediate canvas (example 1 in the spec). However these have two shortcomings:

* Canvas methods are synchronous. Processing large amounts of data, or large numbers of images, can cause "jank" or even lockup the browser entirely while work is done.
* They require a backing canvas, which can add significant additional memory overhead, and in some cases is GPU-accelerated making pixel read/write operations slow as they may have to upload/download from GPU memory.

To avoid these shortcomings, this proposal adds new methods to *asynchronously* convert `HTMLImageElement` and `Blob` to `ImageData`, and `ImageData` to `Blob`, *without* requiring an intermediate canvas.

## Don't images already load asynchronously?

No. Do not be fooled by the `onload` event of images. In all modern browsers this fires when the image has *downloaded*, but not *decoded*. Think of it as really downloading a `Blob` of the image file behind the scenes, and firing `onload` when the AJAX request completes. It only has the compressed image data. The first time you *use* the image (such as displaying it in the DOM, passing it to canvas `drawImage` etc) it will decompress the image data just in time. This is called *lazy loading*, and is an important part of making software systems efficient. If the browser fully decoded the image before `onload`, it would waste CPU time and memory decoding images that may not actually be used at all - so it's actually a good thing it works that way.

As a consequence, if you convert an Image to ImageData via canvas `drawImage`, the browser will *synchronously decode the image* in the `drawImage` call. Since large images can take a long time to decode (sometimes hundreds of milliseconds!) this can easily land you in jank hell.

## So why not add Image.decode() and ondecoded?

It's great if web apps can offload heavy work - such as processing image data - in to a web worker. HTMLImageElement is a DOM element, and workers have no access to the DOM. So we need ways to convert between types like Blob and ImageData that do not depend on Image elements.

## How about an async `getImageData`?

The spec includes a section on some potential alternative solutions, including this one. In this case it's notable that it does not help reduce the memory usage - you still need an image-sized intermediate canvas. Also if there is an asynchronous `getImageData` and `putImageData`, these will race with other synchronous drawing commands like `drawImage`, creating opportunity for bugs that depend on what order operations are really completed.

# So what is the proposal?

The proposal adds two ImageData methods for converting. `ImageData.create()` returns a promise, asynchronously converting either a HTMLImageElement or a Blob to an ImageData. Note no canvas is involved! Further there is a new `toBlob()` method, intended to work similarly to the canvas element's `toBlob()`, asynchronously encoding the ImageData to a compressed format like PNG and returning a Blob representing the file.

Since it's useful to know which image formats the web app can use, it also proposes `canDecodeType` and `canEncodeType`, intended to be the image equivalents of `HTMLMediaElement.canPlayType` and `MediaRecorder.canRecordType` to indicate which formats the browser understands. For example a web app could use these to easily determine if it can encode a WebP file without having to resort to ugly feature detection tricks. Whilst PNG and JPEG are the primary formats used on the web, there are actually a number of formats that have partial support across browsers:

* JPEG-2000
* JPEG-XR
* WebP
* APNG
* Motion JPEG
* Experimental formats like BPG (Better Portable Graphics)
* ...likely other new formats in future

These methods allow a web app to easily display a list of encoding choices, provide guidance on supported formats, etc.

## Future work

Although animated formats like Motion JPEG and APNG were mentioned, this proposal only deals with single individual images. For animated formats it is suggested a new interface be created in future, such as `AnimatedImageData`, which can return the frame count and asynchronously retrieve each frame as a separate ImageData. To also allow encoding animated formats it should also be able to add and remove frames. However this should not have any impact on this proposal since they are essentially arrays of static images with some extra metadata and ImageData can still be used to represent each static image. Animated image support would be useful to complete the feature set for encoding/decoding all image formats though.