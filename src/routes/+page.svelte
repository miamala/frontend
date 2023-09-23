<script lang="ts">
	import { onMount } from 'svelte';

	let mediaRecorder: MediaRecorder | null = null;
	let audioElement: HTMLAudioElement | null = null;
	let canvasElement: HTMLCanvasElement | null = null;
	let audioCtx: AudioContext | null = null;

	let chunks: Array<Blob> = [];
	let recording = false;

	const saveChunk = (event: BlobEvent): void => {
		chunks.push(event.data);
        console.log("Pushed", event.data)
	};

	async function startRecording() {
		if (!mediaRecorder) {
			console.error('MediaRecorder not initialized');
			return;
		}

		mediaRecorder.start();

        console.log(mediaRecorder.state);
        console.log("recorder started");

		recording = true;
		mediaRecorder.ondataavailable = saveChunk;
	}

	async function stopRecording() {
		if (!mediaRecorder) {
			console.error('MediaRecorder not initialized');
			return;
		}

		mediaRecorder.stop();

        console.log(mediaRecorder.state);
        console.log("recorder stopped");
		recording = false;

		mediaRecorder.ondataavailable = null;

        console.log(chunks)

		const blob = new Blob(chunks, {
			type: 'audio/ogg; codecs=opus'
		});

		chunks.length = 0;
		const audioURL = window.URL.createObjectURL(blob);
		audioElement!.src = audioURL;
	}

	async function main() {
		if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
			try {
				const stream = await navigator.mediaDevices.getUserMedia({
					audio: true
				});

				mediaRecorder = new MediaRecorder(stream);
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
			canvasCtx!.strokeStyle = 'rgb(0, 0, 0)';

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

	onMount(() => {
		main();
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

	<div class="flex justify-center py-4">
		<audio controls bind:this={audioElement} />
	</div>
</main>
