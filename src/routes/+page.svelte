<script lang="ts">
	import Button from '$lib/components/ui/button/button.svelte';
	import Input from '$lib/components/ui/input/input.svelte';
	import { GoogleGenerativeAI, SchemaType } from '@google/generative-ai';
	import { onMount } from 'svelte';

	interface Character {
		id: number;
		name: string;
		image: string;
		description: string;
	}

	// List of characters to fight.
	let characters: Character[] = $state([
		{ id: 1, name: "", image: "", description: "" },
		{ id: 2, name: "", image: "", description: "" }
	]);

	let loading = $state(false);
	let apiKey = $state('');
	$effect(() => {
		if (apiKey) {
			localStorage.setItem('apiKey', apiKey);
		}
	});

	let playing = $state(false);
	let playingText = $state('');
	let playingSize = $state(100);

	let showChar1 = $state(true);
	let showChar2 = $state(true);

	function handleFileSelect(event: Event, character: number) {
		const file = (event.target as HTMLInputElement).files?.[0];
		const selectedCharacter = characters.find(c => c.id === character);
		if (file) {
			const reader = new FileReader();
			reader.onload = (event) => {
				const image = event.target?.result as string;
				if (image && selectedCharacter) {
					selectedCharacter.image = image;
				}
			};
			reader.readAsDataURL(file);
		}
	}

	function dataUrltoInlineData(dataUrl: string) {
		return {
			data: dataUrl.split(',')[1],
			mimeType: dataUrl.split(',')[0].split(':')[1].split(';')[0]
		};
	}

	function handleNameChange(event: Event, character: number) {
		const selectedCharacter = characters.find(c => c.id === character);
		if (selectedCharacter) {
			selectedCharacter.name = (event.target as HTMLInputElement).value;
		}
	}

	function handleDescriptionChange(event: Event, character: number) {
		const selectedCharacter = characters.find(c => c.id === character);
		if (selectedCharacter) {
			selectedCharacter.description = (event.target as HTMLInputElement).value;
		}
	}

	async function generateFight() {
		const apiKey = localStorage.getItem('apiKey') || '';
		const genAI = new GoogleGenerativeAI(apiKey);

		const model = genAI.getGenerativeModel({
			model: 'gemini-1.5-flash',
			systemInstruction:
				'You are a judge. You will be given 2 characters, and you will have to judge who would win in a fight. Even though you are a judge, your response should be funny and sarcastic. Each step should contain a statistic and the character who has it better at said stat. The statistic should only be related to the two opponents. YOUR RESPONSE SHOULD CONTAIN AT LEAST 10 STEPS.'
		});

		loading = true;

		const chat = model.startChat({
			generationConfig: {
				responseMimeType: 'application/json',
				responseSchema: {
					type: SchemaType.OBJECT,
					properties: {
						steps: {
							type: SchemaType.ARRAY,
							items: {
								type: SchemaType.OBJECT,
								properties: {
									statistic: {
										type: SchemaType.STRING
									},
									winner: {
										type: SchemaType.INTEGER
									}
								},
								required: ['statistic', 'winner']
							}
						}
					},
					required: ['steps']
				}
			}
		});

		let result;

		try {

			result = await chat.sendMessage([
				...characters.map((character)=>{
					return {
						inlineData: dataUrltoInlineData(character.image as string)
					}
				}),
				...characters.map((character, index)=>{
					return {
						text:`Character ${index + 1}: ${character.name}; Description: ${character.description};\n`
					}
				})
				{
					text: `Character 1: ${char1?.name} and description: ${char1?.description}; 
					Character 2: ${char2?.name} and description: ${char2?.description}
					`
				}
			]);
		} catch (e: any) {
			alert(e.toString());
			loading = false;
		}

		if (!result) return;

		let steps: { steps: { statistic: string; winner: number }[] } = JSON.parse(
			result.response.text()
		);

		let audio = new Audio('/phonk.mp3');

		await new Promise((resolve) => audio.addEventListener('canplaythrough', resolve));
		loading = false;
		audio.play();
		playing = true;
		playingText = `${char1?.name} vs ${char2?.name}`;
		await sleep(5000);

		let char1Score = 0;
		let char2Score = 0;

		for (let i = 0; i < steps.steps.length; i++) {
			const step = steps.steps[i];

			playingText = `${step.statistic}`;

			await sleep(830);
			playingSize = 120;
			if (step.winner === 1) {
				char1Score++;
				playingText = `${char1?.name}\n\n ${char1Score} - ${char2Score}`;
				showChar1 = true;
				showChar2 = false;
			} else {
				char2Score++;
				playingText = `${char2?.name}\n\n ${char1Score} - ${char2Score}`;
				showChar2 = true;
				showChar1 = false;
			}
			await sleep(830);
			playingSize = 120;
			showChar1 = true;
			showChar2 = true;
		}
		if (char1Score > char2Score) {
			playingText = `${char1?.name} wins!`;
		} else if (char2Score > char1Score) {
			playingText = `${char2?.name} wins!`;
		}
		playingText += `\n\n ${char1Score} - ${char2Score}`;
		await sleep(4000);
		window.location.reload();
	}

	function lerp(start: number, end: number, t: number): number {
		return start * (1 - t) + end * t;
	}

	onMount(() => {
		let interval = setInterval(() => {
			playingSize = lerp(playingSize, 100, 0.1);
		});

		apiKey = localStorage.getItem('apiKey') || '';

		return () => clearInterval(interval);
	});

	function sleep(ms: number) {
		return new Promise((resolve) => setTimeout(resolve, ms));
	}
</script>

{#if !playing}
	<div class="flex h-screen w-full flex-col items-center justify-center">
		
		{#each characters as character, index}
			<div class="mb-5 flex flex-row items-center justify-center gap-2.5">
				<div class="flex flex-col items-center justify-center gap-2.5">
					<img class="h-32 w-32" src={character.image} alt={character.name} />
					<Input type="text" placeholder="name" onchange={(e) => handleNameChange(e, index + 1)} />
					<Input type="text" placeholder="description" onchange={(e) => handleDescriptionChange(e, index + 1)} />
					<Input onchange={(e) => handleFileSelect(e, index + 1)} placeholder="guy" type="file" />
				</div>
	
				<img src="/versus.png" alt="versus" class="h-16 w-16" />
	
				<div class="flex flex-col items-center justify-center gap-2.5">
					<img class="h-32 w-32" src={characters[index === 0 ? 1 : 0].image} alt={characters[index === 0 ? 1 : 0].name} />		
					<Input type="text" placeholder="name" onchange={(e) => handleNameChange(e, index === 0 ? 1 : 0)} />
					<Input type="text" placeholder="description" onchange={(e) => handleDescriptionChange(e, index === 0 ? 1 : 0)} />
					<Input onchange={(e) => handleFileSelect(e, index === 0 ? 1 : 0)} placeholder="guy" type="file" />
				</div>
			</div>
		{/each}
		
		<Input class="mt-2.5 w-64" type="password" placeholder="api key" bind:value={apiKey} />
	</div>
	<Button onclick={generateFight} disabled={loading || !apiKey}
			>Generate Fight</Button
	>
{:else}
	<div
		class="flex h-screen w-full flex-row items-center justify-center"
		style="transform: scale({playingSize / 100});"
	>
		{#if showChar1}
			<img src={char1?.image} alt={char1?.name} class="h-screen w-1/2" />
		{/if}
		{#if showChar2}
			<img src={char2?.image} alt={char2?.name} class="h-screen w-1/2" />
		{/if}
	</div>

	<div class="absolute left-0 top-0 z-10 flex h-screen w-full flex-col items-center justify-center">
		<h1
			style="-webkit-text-stroke: 1px black;"
			class="whitespace-pre-line text-center text-6xl font-bold text-white"
		>
			{playingText}
		</h1>
	</div>
{/if}

<style>
	:global(body) {
		overflow: hidden;
	}
</style>
