<script>
	import { onMount } from 'svelte';
	import { badModels } from '$lib/data/badmodels.js';

	export let selectedModel = '';
	let models = [];
	let loading = true;
	let error = null;

	async function fetchModels() {
		loading = true;
		error = null;
		try {
			const response = await fetch('https://stablehorde.net/api/v2/status/models?type=text');
			if (!response.ok) throw new Error('Failed to fetch models');
			const data = await response.json();

			models = data
				.sort((a, b) => b.count - a.count)
				.filter((model) => !badModels.includes(model.name) && model.eta < 100)
				.map((model) => ({
					name: model.name,
					count: model.count,
					eta: model.eta,
					category: model.name.split('/')[0]
				}));

			if (!selectedModel && models.length > 0) {
				selectedModel = models[0].name;
			}
		} catch (e) {
			error = 'Failed to load models. Please try again.';
		} finally {
			loading = false;
		}
	}

	onMount(fetchModels);

	$: modelsByCategory = models.reduce((acc, model) => {
		if (!acc[model.category]) acc[model.category] = [];
		acc[model.category].push(model);
		return acc;
	}, {});

	const generateSearchURL = (model) => {
		const splitModel = model.split('/');
		const name = splitModel[splitModel.length - 1];
		return `https://huggingface.co/models?search=${name}`;
	};
</script>

<div class="card p-4 variant-soft">
	<h2 class="h4 mb-4">Model Selection</h2>

	{#if error}
		<div class="alert variant-filled-error mb-4">
			{error}
			<button class="btn variant-filled-surface" on:click={fetchModels}>Retry</button>
		</div>
	{/if}

	<div class="flex flex-col gap-2 md:flex-row md:items-center">
		<div class="flex-1">
			<select
				class="select w-full"
				bind:value={selectedModel}
				disabled={loading || models.length === 0}
			>
				{#if loading}
					<option value="">Loading models...</option>
				{:else}
					{#each Object.entries(modelsByCategory) as [category, categoryModels]}
						<optgroup label={category}>
							{#each categoryModels as model}
								<option value={model.name}>
									{model.name.split('/')[1] || model.name}
									({model.count} workers, {model.eta}s)
								</option>
							{/each}
						</optgroup>
					{/each}
				{/if}
			</select>
		</div>

		<div class="flex gap-2 items-center">
			<button class="btn variant-filled" on:click={fetchModels} disabled={loading}>
				{#if loading}
					<span class="loading loading-spinner loading-sm" />
				{:else}
					↻
				{/if}
			</button>

			<a
				href={generateSearchURL(selectedModel)}
				target="_blank"
				rel="noopener noreferrer"
				class="btn variant-filled"
			>
				Hugging Face
			</a>
		</div>
	</div>

	<div class="flex gap-2 mt-2 text-sm">
		<span class="badge variant-soft">
			Workers: {models.find((m) => m.name === selectedModel)?.count || '?'}
		</span>
		<span class="badge variant-soft">
			ETA: {models.find((m) => m.name === selectedModel)?.eta}s
		</span>
	</div>
</div>
