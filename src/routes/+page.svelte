<script lang="ts">
	import { onMount } from 'svelte';

	let mediaRecorder: MediaRecorder | null = null;
	let canvasElement: HTMLCanvasElement | null = null;
	let audioCtx: AudioContext | null = null;
	let fileInput: HTMLInputElement | null = null;

	let chunks: Array<Blob> = [];

	let recording = false;
	let loading = false;

	const saveChunk = (event: BlobEvent): void => {
		chunks.push(event.data);
	};

	async function startRecording() {
		if (!mediaRecorder) {
			console.error('MediaRecorder not initialized');
			return;
		}

		recording = true;
		mediaRecorder.start();
	}

	async function stopRecording() {
		if (!mediaRecorder) {
			console.error('MediaRecorder not initialized');
			return;
		}

		recording = false;
		mediaRecorder.stop();
	}

	async function handleStop() {
		loading = true;

		const blob = new Blob(chunks, {
			type: 'audio/wav'
		});

		// Clear audio buffer
		chunks.length = 0;

		const formData = new FormData();
		formData.append('file', blob, 'audio.wav');

		const response = await fetch('http://localhost:1323/upload', {
			method: 'POST',
			body: formData
		});

		const data = await response.json();

		const transaction: Transaction = JSON.parse(data);
		transactions = [...transactions, transaction];

		loading = false;
	}

	async function main() {
		if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
			try {
				const stream = await navigator.mediaDevices.getUserMedia({
					audio: true
				});

				mediaRecorder = new MediaRecorder(stream);

				mediaRecorder.addEventListener('dataavailable', saveChunk);
				mediaRecorder.addEventListener('stop', handleStop);

				visualize(stream);
			} catch (err) {
				console.error(`The following getUserMedia error occurred: ${err}`);
			}
		} else {
			console.log('getUserMedia not supported on your browser!');
		}
	}

	function visualize(stream: MediaStream) {
		if (!audioCtx) {
			audioCtx = new AudioContext();
		}

		if (!canvasElement) {
			console.error('Canvas element not initialized');
			return;
		}

		const canvasCtx = canvasElement!.getContext('2d');

		if (!canvasCtx) {
			console.error('Canvas context not initialized');
			return;
		}

		const source = audioCtx.createMediaStreamSource(stream);

		const analyser = audioCtx.createAnalyser();
		analyser.fftSize = 2048;
		const bufferLength = analyser.frequencyBinCount;
		const dataArray = new Uint8Array(bufferLength);

		source.connect(analyser);
		//analyser.connect(audioCtx.destination);

		draw();

		function draw() {
			if (!canvasElement) {
				return;
			}

			const WIDTH = canvasElement!.width;
			const HEIGHT = canvasElement!.height;

			requestAnimationFrame(draw);

			analyser.getByteTimeDomainData(dataArray);

			canvasCtx?.clearRect(0, 0, WIDTH, HEIGHT);

			canvasCtx!.lineWidth = 2;
			canvasCtx!.strokeStyle = 'rgb(10, 10, 10)';
			canvasCtx!.beginPath();

			let sliceWidth = (WIDTH * 1.0) / bufferLength;
			let x = 0;
			for (let i = 0; i < bufferLength; i++) {
				let v = dataArray[i] / 128.0;
				let y = (v * HEIGHT) / 2;

				if (i === 0) {
					canvasCtx!.moveTo(x, y);
				} else {
					canvasCtx!.lineTo(x, y);
				}

				x += sliceWidth;
			}

			canvasCtx!.lineTo(canvasElement!.width, canvasElement!.height / 2);
			canvasCtx!.stroke();
		}
	}

	interface Transaction {
		amount: number;
		category: string;
		type: 'income' | 'expense';
	}

	let transactions: Transaction[] = [
		{
			amount: 100,
			category: 'Salary',
			type: 'income'
		}
	];

	async function handleSubmit(event: Event) {
		event.preventDefault();
		loading = true;

		const formData = new FormData();

		formData.append('file', fileInput!.files![0]);
		const response = await fetch('http://localhost:1323/upload', {
			method: 'POST',
			body: formData
		});

		const data = await response.json();
		const transaction: Transaction = JSON.parse(data);

		transactions = [...transactions, transaction];
		loading = false;
	}

	onMount(() => {
		main();

		fileInput?.addEventListener('change', handleSubmit);
	});
</script>

<header>
	<div class="navbar bg-base-100">
		<a class="btn btn-ghost normal-case text-xl" href="#">MiaMala</a>
	</div>
</header>

<main>
	<div class="flex justify-center py-4">
		<div class="relative flex h-8 w-8">
			{#if recording}
				<span
					class="animate-ping absolute inline-flex h-full w-full rounded-full bg-red-400 opacity-75"
				/>
			{/if}
			<button
				class="relative inline-flex h-8 w-8 rounded-full bg-red-500"
				title={!recording ? 'Start recording' : 'Stop recording'}
				on:click={!recording ? startRecording : stopRecording}
			/>
		</div>
	</div>

	<div class="flex justify-center">
		<canvas height="60px" bind:this={canvasElement} />
	</div>
	<!-- 
	<div class="flex justify-center py-4">
		<audio controls bind:this={audioElement} />
	</div> -->

	<div class="flex justify-center mb-4">
		<form
			action="http://localhost:1323/upload"
			method="post"
			enctype="multipart/form-data"
			class="flex justify-center gap-4 items-center"
			on:submit={handleSubmit}
		>
			<!-- upload a file -->

			<label class="btn btn-outline" for="file_input"> Upload </label>
			
			{#if loading}
				<!-- prettierignore -->
				<svg viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg" class="text-primary h-8 w-8">
					<path fill="currentColor" d="M12,4a8,8,0,0,1,7.89,6.7A1.53,1.53,0,0,0,21.38,12h0a1.5,1.5,0,0,0,1.48-1.75,11,11,0,0,0-21.72,0A1.5,1.5,0,0,0,2.62,12h0a1.53,1.53,0,0,0,1.49-1.3A8,8,0,0,1,12,4Z">
						<animateTransform attributeName="transform" type="rotate" dur="0.75s" values="0 12 12;360 12 12" repeatCount="indefinite" />
					</path>
				</svg>
			{/if}

			<input class="hidden" id="file_input" type="file" name="file" bind:this={fileInput} />

			<input type="submit" value="Submit" class="btn btn-primary hidden" />
		</form>
	</div>

	<div class="flex flex-col gap-4 items-center">
		{#each transactions as transaction}
			<div class="card w-96 bg-base-100 shadow-xl">
				<div class="card-body flex gap-4">
					<p>
						{#if transaction.type === 'income'}
							<!-- prettier-ignore -->
							<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6 text-green-500">
								<path stroke-linecap="round" stroke-linejoin="round" d="M12 19.5v-15m0 0l-6.75 6.75M12 4.5l6.75 6.75" />
							</svg>
						{:else}
							<!-- prettier-ignore -->
							<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6 text-red-500">
								<path stroke-linecap="round" stroke-linejoin="round" d="M19.5 13.5L12 21m0 0l-7.5-7.5M12 21V3" />
						  	</svg>
						{/if}
					</p>

					<p>{transaction.category}</p>

					<p class="text-lg font-bold">KES {transaction.amount}</p>
				</div>
			</div>
		{/each}
	</div>
</main>
