<script lang="ts">
	import Image, { type SanityImageAsset, type SanityImage } from '$lib/Image.svelte';

	const image = {
		_type: 'image',
		asset: {
			_ref: 'image-120e14321620c478f86def8d28cc460da993eddd-1600x1371-png',
			_type: 'reference'
		}
	} as SanityImageAsset;
</script>

<Image {image} priority width={200} height={100} />
<Image
	{image}
	queryParams={{
		q: 100
	}}
	width={100}
	height={100}
/>
<Image {image} natural />
