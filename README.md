# sanity-svelte-image

[![Latest version](https://img.shields.io/npm/v/sanity-svelte-image?label=version&color=brightGreen&logo=npm)](https://www.npmjs.com/package/sanity-svelte-image)
[![Open issues](https://img.shields.io/github/issues/dennisregalado/sanity-svelte-image)](https://github.com/dennisregalado/sanity-svelte-image/issues)
![Svelte version compatibility](https://img.shields.io/badge/svelte-^5.0.0-orange?logo=svelte)

A well-considered Svelte component for displaying images from Sanity. At a
glance:

- Outputs a single `<img>` tag, no nested DOM structure to mess with
- Zero styling included so you can style it however you want…it's just an `img`
  tag!
- Generates a `srcSet` automatically based on the `width` you specify
- Dynamic `srcSet` factor based on image output width
- Knows _exactly_ what size the image will be and sets `width` and `height`
  attributes accordingly
- Supports `crop` and `hotspot` values from the Sanity Studio
- Automatically crops to the most "interesting" part of the image if the aspect
  ratio changes and no `hotspot` is provided
- Images are _never_ scaled up
- Tiny bundle size with minimal dependencies
- TypeScript support
- Works with SvelteKit and any other Svelte-based framework

**Note**: Low-quality image preview support is not yet implemented but is planned for a future release.

## Quick Start

### Install it:

```sh
npm install sanity-svelte-image
# or
yarn add sanity-svelte-image
# or
pnpm add sanity-svelte-image
```

### Use it:

You can find the full writeup on getting going below, but in the interest of
making it easy to see if this is the thing you are looking for, here's a quick
example of most of what you'll need to know:

**Simplest Case**:

This will render the image out assuming it will be displayed at half its
original width with a srcSet included (multiplies vary based on original image
size):

```svelte
<script lang="ts">
  import { Image } from 'sanity-svelte-image';
  
  let { image } = $props();
</script>

<Image
  {image}
  alt="Demo image"
/>
```

**More full-featured example**:

```svelte
<script lang="ts">
  import { Image } from 'sanity-svelte-image';
  
  let { image, heroImage, thumbnailImage } = $props();
</script>

<!-- Hero image with priority loading -->
<Image
  image={heroImage}
  width={1200}
  height={600}
  mode="cover"
  priority={true}
  alt="Hero image"
  class="w-full h-auto"
/>

<!-- Regular image with custom sizing -->
<Image
  {image}
  width={500}
  height={250}
  mode="cover"
  queryParams={{ sharpen: 30, q: 80 }}
  alt="Featured content"
  class="rounded-lg shadow-md"
  sizes="(min-width: 500px) 500px, 100vw"
/>

<!-- Natural size image (preserves original dimensions) -->
<Image
  image={thumbnailImage}
  natural={true}
  alt="Thumbnail at natural size"
  class="max-w-xs"
/>
```

That's the gist. Read on for more. 👇

## Details

How it works at a glance:

- The image asset reference is parsed to determine the source image dimensions and format
- SVG images get special treatment from the Sanity Image API (they don't support
  params), so they're handled differently
- All other images have `src` and `srcSet` props generated based on the `width`
  and `height` props you pass in (or the image dimensions if you don't pass in a
  width or height)
- The `srcSet` widths depend on the size of the output image and the original
  image; there's logic to avoid wasteful tiny images or giant jumps in size
  between large entries
- Values in the `srcSet` are never duplicated and never upscale the image
- Since we can compute the output dimensions of the image in all cases, the
  `width` and `height` attributes are set automatically to avoid layout shifts
- A few image params are applied by default:
  - `auto=format` - Sanity will use AVIF images if they're supported by the
    browser (note: if you specify `fm` manually, this won't be set)
  - `fit` - if the image aspect ratio isn't changed, this will be set to `max`;
    if the aspect ratio will change it's set to `crop`
  - `q` - the quality is set to 75 by default, but you can override it with the
    `queryParams` prop
- The `loading` attribute will be set to `lazy` if it isn't supplied; use
  `loading="eager"` for images above the fold or set `priority={true}`
- The `alt` attribute will be set to an empty string if it isn't supplied; set
  it if it isn't a decorative image!
- Query params passed to Sanity are all sorted and minimized for improved
  caching and smaller URLs

## Props

This component accepts all standard HTML `img` attributes plus the following Sanity-specific props:

### Required Props

- `image` (SanityImageAsset) — Required - The Sanity image object containing the asset reference and optional crop/hotspot data

### Optional Props

- `mode` ("cover" | "contain") — Optional - Use `cover` to crop the image to
  match the requested aspect ratio (based on `width` and `height`). Use
  `contain` to fit the image to the boundaries provided without altering the
  aspect ratio. Defaults to `"contain"` unless `natural={true}` is set.
- `width` (number) — Optional - The target width of the image in pixels. Only
  used for determining the dimensions of the generated assets, not for layout.
  Use CSS to specify how the browser should render the image instead.
- `height` (number) — Optional - The target height of the image in pixels. Only
  used for determining the dimensions of the generated assets, not for layout.
  Use CSS to specify how the browser should render the image instead.
- `natural` (boolean) — Optional - When true, uses the original image dimensions
  and sets mode to "contain". Defaults to `false`.
- `priority` (boolean) — Optional - When true, sets `loading="eager"`, 
  `fetchpriority="high"`, and `decoding="sync"` for above-the-fold images.
  Defaults to `false`.
- `baseUrl` (string) — Optional - Custom base URL for the Sanity CDN. If not
  provided, uses environment variables `PUBLIC_SANITY_PROJECT_ID` and 
  `PUBLIC_SANITY_DATASET`.
- `queryParams` (object) — Optional - An object of query parameters to pass to
  the Sanity Image API. See the
  [Sanity Image API documentation](https://www.sanity.io/docs/image-urls) for a
  list of available options.

### HTML Override Props

Since some props conflict with native `<img>` attributes, these prefixed props
are provided:

- `htmlWidth` (number) — Override the computed width attribute
- `htmlHeight` (number) — Override the computed height attribute  
- `htmlId` (string) — Set the `id` attribute (since `id` is used for Sanity image ID)

## Setup

### Environment Variables

Add your Sanity project configuration to your environment variables:

```env
PUBLIC_SANITY_PROJECT_ID=your-project-id
PUBLIC_SANITY_DATASET=production
```

For SvelteKit, these should be in your `.env` file and prefixed with `PUBLIC_` to be available on the client side.

## Tips

### Choosing the right `mode`

If you are providing only one dimension (`width` or `height`, but not both), it
doesn't matter since the behavior will be the same.

- **Contain** mode will treat the dimensions you provide as boundaries, resizing
  the image to fit inside of them. The output image will match the aspect ratio
  of the original image (i.e., no cropping will occur).
- **Cover** mode will treat the dimensions you provide as a container, resizing
  the image to completely fill the dimensions. The output image will match the
  aspect ratio of the dimensions you provide.

### Using `priority` and `natural` props

**Priority images** are for above-the-fold content that should load immediately:

```svelte
<!-- Hero image that should load first -->
<Image
  image={heroImage}
  width={1200}
  height={600}
  priority={true}
  alt="Page hero"
  class="w-full h-auto"
/>
```

**Natural sizing** preserves the original image dimensions without any resizing:

```svelte
<!-- Profile picture at its natural size -->
<Image
  image={avatarImage}
  natural={true}
  alt="User avatar"
  class="rounded-full max-w-20"
/>

<!-- Logo that should maintain its exact proportions -->
<Image
  image={logoImage}
  natural={true}
  alt="Company logo"
  class="h-12 w-auto"
/>
```

### Styling your images

I recommend setting something like the following Tailwind classes for images in your project,
then overriding styles as needed. This will ensure images act like block-level
elements with infinitely scalable contents even with the `width` and `height`
attributes set:

```css
/* Add to your global CSS */
img {
  @apply block max-w-full w-full h-auto;
}
```

Or apply directly to images:
```svelte
<Image {image} class="block max-w-full w-full h-auto" alt="My image" />
```

Here's an example of how that works when using a 3-column grid:

```svelte
<script lang="ts">
  import { Image } from 'sanity-svelte-image';
  
  let { images } = $props();
</script>

<div class="grid grid-cols-3 gap-4 max-w-screen-xl px-5 mx-auto">
  {#each images as image (image._id)}
    <div>
      <Image
        {image}
        width={390}
        sizes="(min-width: 1240px) 390px, calc((100vw - 40px - 30px) / 3)"
        alt="Gallery image"
        class="block max-w-full w-full h-auto"
      />
    </div>
  {/each}
</div>
```

If you need these images to all match in height, switch to `cover` mode:

```svelte
<Image
  {image}
  width={390}
  height={260}
  mode="cover"
  sizes="(min-width: 1240px) 390px, calc((100vw - 40px - 30px) / 3)"
  alt="Gallery image"
/>
```

### Background images with Svelte

Using the Image component for background images:

```svelte
<script lang="ts">
  import { Image } from 'sanity-svelte-image';
  
  let { backgroundImage } = $props();
</script>

<section class="relative py-24">
  <Image
    image={backgroundImage}
    width={1440}
    class="absolute inset-0 w-full h-full object-cover select-none z-0"
    alt=""
  />
  
  <div class="relative z-10">
    <h1 class="text-4xl font-bold">Your big hero copy</h1>
    <a href="/signup/" class="inline-block mt-4 px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700">
      Get started
    </a>
  </div>
</section>
```


### Fetching data from Sanity via GROQ

If you're using Sanity's GROQ query language to fetch data, here is how I
recommend fetching the image fields:

```groq
// For a simple image field
image {
  ...
}

// If you need the asset reference directly
image {
  asset->{
    _id,
    _ref
  },
  hotspot { x, y },
  crop {
    bottom,
    left,
    right,
    top,
  }
}
```

## Type Definitions

The component exports the following TypeScript types:

```typescript
import type { 
  SanityImage,                        // Main component props type
  SanityImageAsset                    // Sanity image object type
} from 'sanity-svelte-image';

// You can also import the baseUrl helper
import { baseUrl } from 'sanity-svelte-image';
```

## Roadmap

- [ ] Low-quality image preview (LQIP) support
- [ ] `<picture>` element support with `<source>` tag rendering
- [ ] Blur-up transitions

## License

MIT License - see LICENSE file for details.

## Contributing

Issues and pull requests are welcome! Please check the existing issues before creating a new one.

## Credits

This component is inspired by and based on the excellent [sanity-image](https://github.com/coreyward/sanity-image) React component by [Corey Ward](https://github.com/coreyward). The Svelte implementation provides the same powerful features adapted for the Svelte ecosystem.