<script>
	import { onMount } from 'svelte';
	import { badModels } from '$lib/data/badModels.js';

	let models = [];
	export let selectedModel = '';

	onMount(async () => {
		const response = await fetch('https://horde.koboldai.net/api/v2/status/models?type=text');
		const data = await response.json();
		const sortedModels = data
			.sort((a, b) => b.count - a.count)
			.filter((model) => !badModels.includes(model.name) && model.eta < 100);

		models = sortedModels.map((model) => ({
			name: model.name,
			count: model.count,
			eta: model.eta
		}));
		selectedModel = models[0].name;
	});
</script>

<h4 class="h4 mb-2">Select Model</h4>
<div class="flex gap-2">
	<select class="select" bind:value={selectedModel}>
		{#each models as model}
			<option value={model.name}>{model.name} (QTY: {model.count}, ETA: {model.eta}s)</option>
		{/each}
	</select>
	<a
		href={`https://huggingface.co/models?search=${selectedModel}`}
		target="_blank"
		rel="noopener noreferrer"
		class="btn variant-filled">Hugging Face</a
	>
</div>
