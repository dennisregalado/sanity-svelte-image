<script lang="ts" module>
	import { PUBLIC_SANITY_PROJECT_ID, PUBLIC_SANITY_DATASET } from '$env/static/public';
	import { type ImageQueryInputs } from '@sanity-image/url-builder';
	import type { HTMLImgAttributes } from 'svelte/elements';

	if (!PUBLIC_SANITY_PROJECT_ID) {
		throw new Error(
			'Missing required `PUBLIC_SANITY_PROJECT_ID` environment variable for <SanityImage>.'
		);
	} else if (!PUBLIC_SANITY_DATASET) {
		throw new Error(
			'Missing required `PUBLIC_SANITY_DATASET` environment variable for <SanityImage>.'
		);
	}

	export const baseUrl = `https://cdn.sanity.io/images/${PUBLIC_SANITY_PROJECT_ID}/${PUBLIC_SANITY_DATASET}/`;

	declare const internalGroqTypeReferenceTo: unique symbol;

	export type SanityImageHotspot = {
		_type: 'sanity.imageHotspot';
		x?: number;
		y?: number;
		height?: number;
		width?: number;
	};

	export type SanityImageCrop = {
		_type: 'sanity.imageCrop';
		top?: number;
		bottom?: number;
		left?: number;
		right?: number;
	};

	export type SanityImageAsset = {
		asset?: {
			_ref: string;
			_type: 'reference';
			_weak?: boolean;
			[internalGroqTypeReferenceTo]?: 'sanity.imageAsset';
		};
		hotspot?: SanityImageHotspot;
		crop?: SanityImageCrop;
		_type: 'image';
	};

	/**
	 * These props tell `SanityImage` about your Sanity project in order to build
	 * the full URLs to your images from their `_id`.
	 */
	export type SanityImageConfigurationProps = {
		/**
		 * The base URL for the Sanity CDN. If not provided, the `projectId` and
		 * `dataset` props will be used to construct the URL.
		 */
		baseUrl?: string;
	};

	/**
	 * SanityImage accepts some props that conflict with native `<img>` attributes.
	 * In order to set these attributes on the rendered element, these
	 * `html`-prefixed props are provided.
	 */
	export type SanityImageAttributeOverrideProps = {
		/**
		 * Passed through to the rendered element as `height`, overriding the default
		 * behavior of setting the `height` property automatically based on the
		 * computed output image dimensions.
		 */
		htmlHeight?: number;

		/**
		 * Passed through to the rendered element as `width`, overriding the default
		 * behavior of setting the `width` property automatically based on the
		 * computed output image dimensions.
		 */
		htmlWidth?: number;

		/**
		 * Passed through to the rendered element as `id`.
		 *
		 * The `id` prop is used to specify the Sanity Image ID, so this is the only
		 * way to set the `id` attribute on the rendered element.
		 */
		htmlId?: string;
	};

	/**
	 * Props for the `SanityImage` component itself. Does not include any props that
	 * are used to build the underlying URLs.
	 */
	export type SanityImageComponentProps = {
		preview?: string;
	};

	/**
	 * All image-rendering props.
	 */
	export type BaseImageProps = SanityImageAttributeOverrideProps &
		SanityImageComponentProps &
		Omit<ImageQueryInputs, 'id'> & {
			// Custom props
			image: SanityImageAsset;
			id?: string;
			priority?: boolean;
			natural?: boolean;
			tag?: 'img' | 'source';
		};

	/**
	 * Configuration props + image-rendering props.
	 */
	export type FullImageProps = BaseImageProps & SanityImageConfigurationProps;

	export type SanityImage = HTMLImgAttributes & FullImageProps;
</script>

<script lang="ts">
	import { buildSrc, buildSrcSet, buildSvgAttributes } from '@sanity-image/url-builder';

	let {
		// Sanity url
		baseUrl: propBaseUrl,

		// Custom props
		image,
		priority = false,
		natural = false,
		tag = 'img',

		// Image definition data
		id,
		hotspot,
		crop,
		width,
		height,
		mode = natural ? 'contain' : 'cover',

		// Data for LQIP (preview image)
		preview,

		// Native-behavior overrides
		htmlWidth,
		htmlHeight,
		htmlId,

		// Image query string params
		queryParams,

		// Any remaining props are passed through to the rendered component
		...rest
	}: SanityImage = $props();

	let resolvedBaseUrl = $derived(propBaseUrl ?? baseUrl);

	let resolvedId = $derived(image?.asset?._ref || id);

	const isSvg = $derived(resolvedId?.endsWith('-svg'));

	const baseProps: Record<string, unknown> = $derived({
		alt: rest.alt ?? '',
		loading: rest.loading ?? 'lazy',
		id: htmlId,
		...rest
	});

	// Create default src and build srcSet
	const srcParams = $derived({
		baseUrl: resolvedBaseUrl,
		id: resolvedId!,
		crop,
		hotspot,
		width,
		height,
		mode,
		queryParams
	});

	const { src, ...outputDimensions } = $derived(resolvedId ? buildSrc(srcParams) : { src: '', width: 0, height: 0 });
	const srcset = $derived(resolvedId ? buildSrcSet(srcParams).join(', ') : '');

	const loadingProps = $derived({
		loading: priority ? ('eager' as const) : ('lazy' as const),
		fetchpriority: priority ? ('high' as const) : ('auto' as const),
		decoding: priority ? ('sync' as const) : ('async' as const)
	});

	const svgProps = $derived.by(() => {
		if (!isSvg) return;

		// Sanity ignores all transformations for SVGs, so we can just render the
		// component without passing a query string and without doing anything for
		// the preview.
		const baseAttributes: Record<string, unknown> = buildSvgAttributes({
			id: resolvedId!,
			baseUrl
		});

		// If this is a <source> element, we need to set the `srcSet` attribute and not
		// the `src` attribute, otherwise it will be ignored in <picture> elements.
		if (tag === 'source') {
			baseAttributes.srcSet = baseAttributes.src;
			delete baseAttributes.src;
		}

		return {
			...baseAttributes,
			...baseProps
		};
	});

	const imageProps = $derived({
		...baseProps,
		...outputDimensions,
		...loadingProps,
		src,
		srcset,
		width: htmlWidth ?? outputDimensions.width,
		height: htmlHeight ?? outputDimensions.height
	});
</script>

<svelte:head>
	{#if priority && !isSvg && !preview}
		<link rel="preload" as="image" imagesrcset={srcset} fetchpriority="high" />
	{/if}
</svelte:head>

{#if isSvg}
	<img {...svgProps} />
{:else}
	<svelte:element this={tag} {...imageProps} />
{/if}
