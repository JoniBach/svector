<script>
	import Canvas from '$lib/Canvas.svelte';
	export let data;
	let file = '';
	let id;

	const actions = [
		{ type: 'upload' },
		{ type: 'download' },
		{ type: 'text' },
		{ type: 'shape' },
		{ type: 'vector' },
		{
			type: 'custom',
			name: 'Generate Image',
			icon: 'mdi:sparkles',
			action: () => generateImage()
		}
	];

	async function handleSave(event) {
		// Trigger form action save
		console.log('Save event triggered', event.detail);
	}

	async function generateImage() {
		const promptText = prompt('Enter a prompt for image generation:');
		if (promptText !== null) {
			// eventDispatcher('generateImage', { prompt: promptText });
			alert('Generate Image', promptText);
		}
	}
</script>

<div class="create-page-container">
	<Canvas on:save={handleSave} {file} {id} {actions} />
</div>

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		font-family: 'Roboto', sans-serif;
		box-sizing: border-box;
	}
</style>
