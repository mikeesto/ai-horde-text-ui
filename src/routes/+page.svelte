<script lang="ts">
	import ModelSelector from '$lib/components/ModelSelector.svelte';
	import { tick } from 'svelte';
	import { Accordion, AccordionItem } from '@skeletonlabs/skeleton';

	type ChatMessage = {
		message: string;
		from: 'user' | 'assistant';
		timestamp: Date;
	};

	let chats: ChatMessage[] = [];
	let userMessage = '';
	let selectedModel = '';
	let systemPrompt =
		'You are an AI that follows instructions extremely well. Help as much as you can.';
	let chatSection: HTMLElementTagNameMap['section'];
	let loading = false;
	let error: string | null = null;

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
		error = null;
		loading = true;

		const newMessage: ChatMessage = {
			message: userMessage,
			from: 'user',
			timestamp: new Date()
		};

		chats = [...chats, newMessage];
		scrollToBottom(chatSection);

		try {
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
			}).then((res) => {
				if (!res.ok) throw new Error('Failed to send message');
				return res.json();
			});

			userMessage = '';

			const interval = setInterval(async () => {
				try {
					const { done, generations } = await fetch(
						`https://stablehorde.net/api/v2/generate/text/status/${id}`
					).then((res) => res.json());

					if (done) {
						clearInterval(interval);
						chats = [
							...chats,
							{
								message: generations[0].text,
								from: 'assistant',
								timestamp: new Date()
							}
						];
						loading = false;
						scrollToBottom(chatSection);
					}
				} catch (e) {
					clearInterval(interval);
					error = 'Failed to get response';
					loading = false;
				}
			}, 1000);
		} catch (e) {
			error = 'Failed to send message';
			loading = false;
		}
	}

	function resetChat() {
		chats = [];
	}
</script>

<div class="mt-2 md:max-w-2xl mx-auto">
	<div class="flex flex-col max-h-screen">
		<h1 class="text-2xl">AI Horde Text UI</h1>
		<p class="text-sm opacity-80 mb-4 mt-2">
			The <a href="https://stablehorde.net/" class="text-orange-200">AI Horde</a> is a crowdsourced distributed
			cluster of text generation workers.
		</p>

		<ModelSelector bind:selectedModel />

		<div class="flex items-center gap-2 mt-4">
			<Accordion class="flex-1">
				<AccordionItem>
					<svelte:fragment slot="lead">⚙️</svelte:fragment>
					<svelte:fragment slot="summary">System Prompt</svelte:fragment>
					<svelte:fragment slot="content">
						<textarea
							class="textarea resize-none w-full"
							rows="2"
							placeholder="Enter a system prompt..."
							bind:value={systemPrompt}
						/>
					</svelte:fragment>
				</AccordionItem>
			</Accordion>
			<button class="btn variant-soft-error" on:click={resetChat}>Reset Chat</button>
		</div>

		{#if error}
			<div class="alert variant-filled-error my-2">{error}</div>
		{/if}

		<section
			class="space-y-4 mt-4 overflow-scroll mb-[100px] hide-scrollbar"
			bind:this={chatSection}
		>
			{#each chats as chat}
				<div
					class="card p-4 {chat.from === 'user'
						? 'variant-soft-secondary ml-8'
						: 'variant-soft-primary mr-8'}"
				>
					<div class="flex justify-between items-start gap-2">
						<div class="flex-1 break-words">
							{chat.message}
						</div>
						<div class="text-xs opacity-50">
							{chat.timestamp.toLocaleTimeString()}
						</div>
					</div>
				</div>
			{/each}
			{#if loading}
				<div class="card p-4 variant-soft-primary mr-8">
					<div class="flex items-center gap-2">
						<div class="loading loading-spinner loading-sm" />
						<span>Thinking...</span>
					</div>
				</div>
			{/if}
		</section>

		<form
			on:submit|preventDefault={sendMessage}
			class="absolute bottom-0 mb-6 max-w-2xl w-full input-group input-group-divider grid-cols-[1fr_auto] rounded-container-token bg-surface-100-800-token"
		>
			<textarea
				bind:value={userMessage}
				on:input={handleInput}
				on:keydown={(e) => {
					if (e.key === 'Enter' && !e.shiftKey) {
						e.preventDefault();
						sendMessage();
					}
				}}
				class="bg-transparent border-0 ring-0 resize-none p-3"
				name="prompt"
				id="prompt"
				placeholder="Write a message... (Shift+Enter for new line)"
				rows="1"
			/>
			<button class="variant-filled-primary" disabled={loading || !userMessage || !selectedModel}>
				{loading ? 'Sending...' : 'Send'}
			</button>
		</form>
	</div>
</div>
