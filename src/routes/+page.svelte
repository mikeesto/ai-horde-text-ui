<script lang="ts">
	import ModelSelector from '$lib/components/ModelSelector.svelte';
	import { tick } from 'svelte';

	let chats: { message: string; from: 'user' | 'assistant' }[] = [];
	let userMessage = '';
	let selectedModel = '';
	let systemPrompt =
		'You are an AI that follows instructions extremely well. Help as much as you can.';
	let chatSection: HTMLElementTagNameMap['section'];
	let loading = false;

	function handleInput(event: Event) {
		const target = event.target as HTMLTextAreaElement;
		target.style.height = 'auto';
		target.style.height = target.scrollHeight + 'px';
	}

	const scrollToBottom = async (node: HTMLElementTagNameMap['section']) => {
		await tick();
		node.scroll({ top: node.scrollHeight, behavior: 'smooth' });
	};

	function generatePrompt() {
		// Note: This is a very generic prompt template, not specific to any model.
		let prompt = `System: ${systemPrompt}`;
		for (const chat of chats) {
			if (chat.from === 'user') {
				prompt += `\nUser: ${chat.message}`;
			} else {
				prompt += `\nAssistant: ${chat.message}`;
			}
		}
		return prompt;
	}

	async function sendMessage() {
		if (loading || !userMessage || !selectedModel) return;

		loading = true;

		chats = [...chats, { message: userMessage, from: 'user' }];
		scrollToBottom(chatSection);

		const { id } = await fetch('https://stablehorde.net/api/v2/generate/text/async', {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json',
				apiKey: '0000000000'
			},
			body: JSON.stringify({
				models: [selectedModel],
				prompt: generatePrompt(),
				params: {
					max_context_length: 2048,
					max_length: 512
				}
			})
		}).then((res) => res.json());

		userMessage = '';

		const interval = setInterval(async () => {
			const { done, generations } = await fetch(
				`https://stablehorde.net/api/v2/generate/text/status/${id}`
			).then((res) => res.json());

			if (done) {
				clearInterval(interval);
				chats = [...chats, { message: generations[0].text, from: 'assistant' }];
				loading = false;
				scrollToBottom(chatSection);
			}
		}, 1000);
	}
</script>

<div class="mt-2 md:max-w-2xl mx-auto">
	<div class="flex flex-col max-h-screen">
		<ModelSelector bind:selectedModel />
		<label class="label">
			<h4 class="h4 mb-2 mt-2">System Prompt</h4>
			<textarea
				class="textarea resize-none"
				rows="2"
				placeholder="Enter a system prompt..."
				bind:value={systemPrompt}
			/>
		</label>
		<form
			on:submit|preventDefault={sendMessage}
			class="absolute bottom-0 mb-6 max-w-2xl input-group input-group-divider grid-cols-[1fr_auto] rounded-container-token"
		>
			<textarea
				bind:value={userMessage}
				on:input={handleInput}
				on:keydown={(e) => {
					if (e.key === 'Enter') {
						e.preventDefault();
						sendMessage();
					}
				}}
				class="bg-transparent border-0 ring-0 resize-none"
				name="prompt"
				id="prompt"
				placeholder="Write a message..."
				rows="1"
			/>
			<button class="variant-filled-primary" disabled={loading}>Send</button>
		</form>
		<h4 class="h4 mb-2 mt-2">Chat</h4>
		<section
			class="space-y-4 mt-4 overflow-scroll mb-[100px] hide-scrollbar"
			bind:this={chatSection}
		>
			{#each chats as chat}
				{#if chat.from === 'user'}
					<div class="card p-4 variant-soft-secondary">
						{chat.message}
					</div>
				{:else}
					<div class="card p-4 variant-soft-primary">
						{chat.message}
					</div>
				{/if}
			{/each}
		</section>
	</div>
</div>
