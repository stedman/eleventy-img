# eleventy-img

Low level utility to perform build-time image transformations.

## Features

* Optimize and resize images, automatically.
  * Can output multiple image sizes.
  * Keeps original image aspect ratios intact.
  * Never upscales images larger than original size.
* Output multiple image formats.
  * The [sharp](https://sharp.pixelplumbing.com/) image processor supports `jpeg`, `png`, and `webp`.
  * Incoming `gif` and `svg` images are converted to `png`.
* Cache remote images locally using [eleventy-cache-assets](https://github.com/11ty/eleventy-cache-assets).
  * Use "local" images in your HTML to prevent broken image URLs.
  * Manage the [cache duration](https://github.com/11ty/eleventy-cache-assets#change-the-cache-duration).
* Get image output data (see [sample return object](#sample-return-object)).
  * Use the image dimension values to set the `width` and `height` attributes on `<img>` elements for [proper aspect ratio mapping](https://developer.mozilla.org/en-US/docs/Web/Media/images/aspect_ratio_mapping).
* Manage image processing concurrency.

## Installation

```
npm install @11ty/eleventy-img
```

## Usage

```js
/* .eleventy.js */

const Image = require("@11ty/eleventy-img");
module.exports = function(eleventyConfig) {
  eleventyConfig.addJavaScriptFunction("myImagePromise", function(src, alt, options) {
    // returns Promise
    return Image(src, options);
  });

  eleventyConfig.addJavaScriptFunction("myImage", async function(src, alt, options) {
    let stats = await Image(src, options);
    let props = stats['jpeg'].pop();
    // returns String
    return `<img src="${props.url} alt="${alt}">`;
  });
};
```

### Options List

Defaults values are shown:

```js
{
  // array of image widths
  widths: [null],
  // widths: [null]      // output original image width
  // widths: [200]       // output 200px maxwidth
  // widths: [200, null] // output 200px and original width

  // output image formats
  formats: ["webp", "jpeg"], // also supported by sharp: "png"

  dir: {
    // image directory to use in img src
    imgSrc: "/img/", // <img src="/img/MY_IMAGE.png">
    // project-relative directory to write the image files
    output: "./", // ./img/
    // ^ note that the actual file system path is a concatenation of dir.output and dir.imgSrc
  },

  // eleventy-cache-assets options (available in eleventy-img 0.3.0+)
  cacheOptions: {
    // if a remote image URL, this is the amount of time before it fetches a fresh copy
    duration: "1d",
    // project-relative path to the cache directory
    directory: ".cache",

    removeUrlQueryParams: false,
  },

  cacheDuration: "1d", // deprecated, use cacheOptions above
}
```

See all [relevant `eleventy-cache-assets` options in its documentation](https://github.com/11ty/eleventy-cache-assets/blob/master/README.md#options).

### Image Processing Concurrency

```js
const Image = require("@11ty/eleventy-img");
Image.concurrency = 4; // default is 10
```

## Examples

> NOTE: The examples below use the [Nunjucks](https://www.11ty.dev/docs/languages/nunjucks/#asynchronous-shortcodes) `async` shortcodes (the [JavaScript](https://www.11ty.dev/docs/languages/javascript/#asynchronous-javascript-template-functions) and [Liquid](https://www.11ty.dev/docs/languages/liquid/#asynchronous-shortcodes) template engines are async by default).

### Specify Image Source and Output Directories

#### Input for Specified Directories

```js
const Image = require("@11ty/eleventy-img");
module.exports = function(eleventyConfig) {
  // works also with addLiquidShortcode or addJavaScriptFunction
  eleventyConfig.addNunjucksAsyncShortcode("myImage", async function(src, alt, outputFormat = "jpeg") {
    if(alt === undefined) {
      // You bet we throw an error on missing alt (alt="" works okay)
      throw new Error(`Missing \`alt\` on myImage from: ${src}`);
    }

    let stats = await Image(src, {
      formats: [outputFormat],
      dir: {
        imgSrc: "/assets/img/",
        output: "./_site/",
      },
    });
    let props = stats[outputFormat].pop();

    return `<img src="${props.src}" width="${props.width}" height="${props.height}" alt="${alt}">`;
  });
};
```

```html
<!-- src/index.njk -->

<div>
  {% myImage "./src/images/cat.jpg", "photo of my cat" %}
</div>
<div>
  {% myImage "https://my_site.com/assets/img/dog.jpg", "photo of my dog" %}
</div>
```

#### Output for Specified Directories

```sh
# project file system

├─ .eleventy.js
┊
├─ _site
│  ├─ assets
│  │  ┊
│  │  ├─ img
│  │  │  ├─ 2311v21.jpeg
│  │  │  └─ 3d00b40.jpeg
┊  ┊  ┊
```

```html
<!-- dist/index.html -->

<div>
  <img src="/assets/img/3d00b40.jpeg" width="1200" height="750" alt="photo of my cat">
</div>
<div>
  <img src="/assets/img/2311v21.jpeg" width="1200" height="750" alt="photo of my dog">
</div>
```

### Specify Output Width

#### Inputs for Specified Width

```js
/* .eleventy.js */

const Image = require("@11ty/eleventy-img");
module.exports = function(eleventyConfig) {
  // works also with addLiquidShortcode or addJavaScriptFunction
  eleventyConfig.addNunjucksAsyncShortcode("myImage", async function(src, alt, outputFormat = "jpeg") {
    if(alt === undefined) {
      // You bet we throw an error on missing alt (alt="" works okay)
      throw new Error(`Missing \`alt\` on myImage from: ${src}`);
    }

    let stats = await Image(src, {
      formats: [outputFormat],
      widths: [600],
    });
    let props = stats[outputFormat].pop();

    return `<img src="${props.url}" width="${props.width}" height="${props.height}" alt="${alt}">`;
  });
};
```

```html
<!-- src/index.njk -->

<div>
  {% myImage "./src/images/cat.jpg", "photo of my cat" %}
</div>
<div>
  {% myImage "https://my_site.com/assets/img/dog.jpg", "photo of my dog" %}
</div>
```

#### Output for Specified Width

```sh
# project file system

├─ .eleventy.js
┊
├─ img
│  ├─ 2311v21-600.jpeg
│  └─ 3d00b40-600.jpeg
┊
```

```html
<!-- dist/index.html -->

<div>
  <img src="/img/3d00b40-600.jpeg" width="600" height="375" alt="photo of my cat">
</div>
<div>
  <img src="/img/2311v21-600.jpeg" width="600" height="375" alt="photo of my dog">
</div>
```

### Specify Multi-Format, Multi-Size Responsive Images using `<picture>`

#### Inputs for Responsive Images

```js
/* .eleventy.js */

const Image = require("@11ty/eleventy-img");
module.exports = function(eleventyConfig) {
  // works also with addLiquidShortcode or addJavaScriptFunction
  eleventyConfig.addNunjucksAsyncShortcode("myResponsiveImage", async function(src, alt, outputFormats = "jpeg") {
    if(alt === undefined) {
      // You bet we throw an error on missing alt (alt="" works okay)
      throw new Error(`Missing \`alt\` on myResponsiveImage from: ${src}`);
    }

    let sizes = "100vw"; // Make sure you customize this!
    // Note: the order of outputFormats matters, preferred formats should be listed first
    let formats = outputFormats.split(",").map(txt => txt.trim());

    let stats = await Image(src, {
      formats,
      widths: [400, 800, null],
    });

    let sourceBlock = Object.values(stats).map((imageFormat) => `<source
      type="image/${imageFormat[0].format}"
      srcset="${imageFormat.map((entry) => `${entry.url} ${entry.width}w`).join(', ')}"
      sizes="${sizes}">`)
      .join('\n');
    // Default output format should be listed last
    let defaultSrc = stats[formats.slice(-1)][0];
    let imgBlock = `<img
      src="${defaultSrc.url}"
      width="${defaultSrc.width}"
      height="${defaultSrc.height}"
      alt="${alt}">`;

    // Iterate over formats and widths
    return `<picture>
        ${sourceBlock}
        ${imgBlock}
      </picture>`;
    });
};
```

```html
<!-- index.njk -->

<div>
  {% myResponsiveImage "./src/images/cat.jpg", "photo of my cat", "webp,jpeg" %}
</div>
<div>
  {% myResponsiveImage "https://my_site.com/assets/img/dog.jpg", "photo of my dog", "webp,jpeg" %}
</div>
```

#### Output for Responsive Images

```sh
# project file system

├─ .eleventy.js
┊
├─ img
│  ├─ 2311v21.jpeg
│  ├─ 2311v21.webp
│  ├─ 2311v21-400.jpeg
│  ├─ 2311v21-400.webp
│  ├─ 2311v21-800.jpeg
│  ├─ 2311v21-800.webp
│  ├─ 3d00b40.jpeg
│  ├─ 3d00b40.webp
│  ├─ 3d00b40-400.jpeg
│  ├─ 3d00b40-400.webp
│  ├─ 3d00b40-800.jpeg
│  └─ 3d00b40-800.webp
┊
```

```html
<!-- dist/index.html -->

<div>
  <picture>
    <source type="image/jpeg" srcset="/img/3d00b40-400.jpeg 400w, /img/3d00b40-800.jpeg 800w, /img/3d00b40.jpeg 1200w" sizes="100vw">
    <source type="image/webp" srcset="/img/3d00b40-400.webp 400w, /img/3d00b40-800.webp 800w, /img/3d00b40.webp 1200w" sizes="100vw">
    <img src="/img/3d00b40-400.jpeg" width="400" height="250" alt="photo of my cat">
  </picture>
</div>
<div>
  <picture>
    <source type="image/jpeg" srcset="/img/2311v21-400.jpeg 400w, /img/2311v21-800.jpeg 800w, /img/2311v21.jpeg 1200w" sizes="100vw">
    <source type="image/webp" srcset="/img/2311v21-400.webp 400w, /img/2311v21-800.webp 800w, /img/2311v21.webp 1200w" sizes="100vw">
    <img src="/img/2311v21-400.jpeg" width="400" height="250" alt="photo of my dog">
  </picture>
</div>
```

## Sample return object

Use this object to generate your responsive image markup.

```js
{
  webp: [
    {
      format: "webp",
      width: 1280,
      height: 853,
      sourceType: "image/webp",
      url: "/img/9b186f9b.webp",
      srcset: "/img/9b186f9b.webp 1280w",
      outputPath: "img/9b186f9b.webp",
      size: 213802
    }, {
      format: "webp",
      width: 350,
      height: 233,
      sourceType: "image/webp",
      url: "/img/9b186f9b-350.webp",
      srcset: "/img/9b186f9b-350.webp 350w",
      outputPath: "img/9b186f9b-350.webp",
      size: 27288
    }
  ],
  jpeg: [
    {
      format: "jpeg",
      width: 1280,
      height: 853,
      sourceType: "image/jpg",
      url: "/img/9b186f9b.jpeg",
      srcset: "/img/9b186f9b.jpeg 1280w",
      outputPath: "img/9b186f9b.jpeg",
      size: 276231
    }, {
      format: "jpeg",
      width: 350,
      height: 233,
      sourceType: "image/jpg",
      url: "/img/9b186f9b-350.jpeg",
      srcset: "/img/9b186f9b-350.jpeg 350w",
      outputPath: "img/9b186f9b-350.jpeg",
      size: 29101
    }
  ]
}
```
