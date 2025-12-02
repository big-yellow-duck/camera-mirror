<script lang="ts">
	import { onMount, onDestroy } from 'svelte';

	let videoElement = $state<HTMLVideoElement>();
	let audioContext: AudioContext | undefined;
	let analyser: AnalyserNode | undefined;
	let stream: MediaStream;
	let audioElement = $state<HTMLAudioElement>();
	let isStreaming = $state(false);
	let hasPermission = $state(false);
	let errorMessage = $state('');
	let isFullscreen = $state(false);
	let micLevel = $state(0);
	let availableMics: MediaDeviceInfo[] = $state([]);
	let selectedMicId = $state('');
	let videoFit = $state<'cover' | 'contain'>('contain'); // 'cover' = fill, 'contain' = maintain aspect ratio
	let audioEnabled = $state(false);
	let audioDelay = $state(0); // Audio delay in milliseconds

	// Canvas for audio level visualization
	let canvasElement: HTMLCanvasElement;
	let animationFrame: number;

	onMount(async () => {
		// Add fullscreen event listener only on client side
		if (typeof document !== 'undefined') {
			document.addEventListener('fullscreenchange', handleFullscreenChange);
		}

		await getAudioDevices();
		// Request camera and microphone permissions on mount
		await requestPermissions();
		startAudioLevelVisualization();
	});

	onDestroy(() => {
		stopStream();
		if (animationFrame && typeof cancelAnimationFrame !== 'undefined') {
			cancelAnimationFrame(animationFrame);
		}
		// Remove fullscreen event listener only on client side
		if (typeof document !== 'undefined') {
			document.removeEventListener('fullscreenchange', handleFullscreenChange);
		}
	});

	// Listen for fullscreen changes to update state correctly
	function handleFullscreenChange() {
		if (typeof document !== 'undefined') {
			isFullscreen = !!document.fullscreenElement;
		}
	}

	async function getAudioDevices() {
		if (typeof navigator === 'undefined') return;

		try {
			const devices = await navigator.mediaDevices.enumerateDevices();
			availableMics = devices.filter((device) => device.kind === 'audioinput');
			if (availableMics.length > 0 && !selectedMicId) {
				selectedMicId = availableMics[0].deviceId;
			}
		} catch (error) {
			console.error('Error getting audio devices:', error);
		}
	}

	async function requestPermissions() {
		if (typeof navigator === 'undefined') return;

		try {
			// Request both camera and microphone
			const constraints: MediaStreamConstraints = {
				video: true,
				audio: selectedMicId ? { deviceId: selectedMicId } : true
			};

			const mediaStream = await navigator.mediaDevices.getUserMedia(constraints);

			stream = mediaStream;
			hasPermission = true;
			errorMessage = '';

			// Set up video stream
			if (videoElement) {
				videoElement.srcObject = stream;
			}

			// Set up audio playback (hear yourself)
			setupAudioPlayback(stream);
			setupAudioAnalyser(stream);
		} catch (error) {
			console.error('Error accessing media devices:', error);
			errorMessage = getErrorMessage(error);
			hasPermission = false;
		}
	}

	function setupAudioPlayback(mediaStream: MediaStream) {
		if (typeof Audio === 'undefined') return;

		try {
			// Create a direct audio element for monitoring
			audioElement = new Audio();
			audioElement.srcObject = mediaStream;
			audioElement.muted = false; // Make sure it's not muted
			audioElement.volume = 0.8; // Set a reasonable volume

			// Try to autoplay
			audioElement
				.play()
				.then(() => {
					audioEnabled = true;
				})
				.catch((error) => {
					console.log('Auto-play was prevented, manual activation needed');
					audioEnabled = false;
				});
		} catch (error) {
			console.error('Error setting up audio playback:', error);
		}
	}

	function toggleAudio() {
		if (!audioElement || !stream) return;

		if (audioEnabled) {
			// Properly stop audio
			audioElement.pause();
			audioElement.currentTime = 0;
			// Clear the audio source to completely stop playback
			audioElement.srcObject = null;
			audioEnabled = false;
		} else {
			// Apply audio delay and set up audio source
			updateAudioDelay();
			audioElement
				.play()
				.then(() => {
					audioEnabled = true;
				})
				.catch((error) => {
					console.error('Error playing audio:', error);
					audioEnabled = false;
				});
		}
	}

	function updateAudioDelay() {
		if (!audioElement || !stream) return;

		// Always stop current playback before applying new delay
		audioElement.pause();
		audioElement.currentTime = 0;

		// Create a delayed audio stream using Web Audio API
		if (audioDelay > 0 && typeof AudioContext !== 'undefined') {
			try {
				// Create a new audio context if needed
				if (!audioContext) {
					audioContext = new AudioContext();
				}

				// Close existing context to avoid conflicts
				if (audioContext.state !== 'closed') {
					audioContext.close();
				}
				audioContext = new AudioContext();

				// Create a delay node
				const delayNode = audioContext.createDelay();
				delayNode.delayTime.value = audioDelay / 1000; // Convert ms to seconds

				// Get the source from the stream
				const source = audioContext.createMediaStreamSource(stream);

				// Create a destination for playback
				const destination = audioContext.createMediaStreamDestination();

				// Connect with delay
				source.connect(delayNode);
				delayNode.connect(destination);
				source.connect(destination); // Also connect without delay for immediate feedback

				// Update audio element with delayed stream
				audioElement.srcObject = destination.stream;
			} catch (error) {
				console.error('Error setting up audio delay:', error);
				// Fallback to no delay
				audioElement.srcObject = stream;
			}
		} else {
			// No delay - use direct stream
			audioElement.srcObject = stream;
		}

		// If audio was enabled, resume playback with new delay
		if (audioEnabled) {
			audioElement.play().catch((error) => {
				console.error('Error resuming audio:', error);
			});
		}
	}

	function setupAudioAnalyser(mediaStream: MediaStream) {
		if (typeof AudioContext === 'undefined') return;

		if (!audioContext) {
			audioContext = new AudioContext();
		}

		const source = audioContext.createMediaStreamSource(mediaStream);
		analyser = audioContext.createAnalyser();
		analyser.fftSize = 256;
		source.connect(analyser);
	}

	function startAudioLevelVisualization() {
		const draw = () => {
			if (!canvasElement) return;
			if (!analyser) return;

			const canvas = canvasElement;
			const canvasContext = canvas.getContext('2d');
			if (!canvasContext) return;

			const bufferLength = analyser.frequencyBinCount;
			const dataArray = new Uint8Array(bufferLength);
			analyser.getByteFrequencyData(dataArray);

			// Clear canvas
			canvasContext.fillStyle = '#1f2937';
			canvasContext.fillRect(0, 0, canvas.width, canvas.height);

			// Calculate average level
			let sum = 0;
			for (let i = 0; i < bufferLength; i++) {
				sum += dataArray[i];
			}
			const average = sum / bufferLength;
			micLevel = average / 255; // Normalize to 0-1

			// Draw audio bars
			const barWidth = (canvas.width / bufferLength) * 2;
			let x = 0;

			for (let i = 0; i < bufferLength; i++) {
				const barHeight = (dataArray[i] / 255) * canvas.height;

				// Color gradient from green to red
				const hue = (1 - dataArray[i] / 255) * 120; // 120 = green, 0 = red
				canvasContext.fillStyle = `hsl(${hue}, 70%, 50%)`;
				canvasContext.fillRect(x, canvas.height - barHeight, barWidth, barHeight);

				x += barWidth + 1;
			}

			if (typeof requestAnimationFrame !== 'undefined') {
				animationFrame = requestAnimationFrame(draw);
			}
		};

		draw();
	}

	function startStream() {
		if (stream && videoElement) {
			videoElement.srcObject = stream;
			videoElement.play();
			isStreaming = true;
		}
	}

	function stopStream() {
		if (videoElement) {
			videoElement.pause();
			videoElement.srcObject = null;
		}

		if (audioElement) {
			audioElement.pause();
			audioElement.currentTime = 0;
			audioElement.srcObject = null;
		}

		if (stream) {
			stream.getTracks().forEach((track) => track.stop());
		}

		if (audioContext) {
			audioContext.close();
			audioContext = undefined; // Clear the audio context
		}

		isStreaming = false;
		audioEnabled = false;
		micLevel = 0;
	}

	async function switchMicrophone() {
		if (stream) {
			stopStream();
			await requestPermissions();
		}
	}

	function toggleFullscreen() {
		if (typeof document === 'undefined') return;

		const videoContainer = videoElement?.parentElement;

		if (!videoContainer) return;

		if (!document.fullscreenElement) {
			videoContainer.requestFullscreen().catch((err) => {
				console.error(`Error attempting to enable video fullscreen: ${err.message}`);
			});
		} else {
			document.exitFullscreen().catch((err) => {
				console.error(`Error attempting to exit fullscreen: ${err.message}`);
			});
		}
	}

	function getErrorMessage(error: any): string {
		if (error.name === 'NotAllowedError') {
			return 'Camera and microphone access was denied. Please allow access in your browser settings.';
		} else if (error.name === 'NotFoundError') {
			return 'No camera or microphone found. Please connect a device and try again.';
		} else if (error.name === 'NotReadableError') {
			return 'Camera or microphone is already in use by another application.';
		} else {
			return 'Error accessing camera or microphone. Please check your device permissions.';
		}
	}
</script>

<div class="min-h-screen bg-gray-900 text-white p-4">
	<div class="max-w-6xl mx-auto">
		<!-- Header -->
		<div class="flex justify-between items-center mb-6">
			<h1 class="text-3xl font-bold">Camera Mirror</h1>
			<button
				onclick={toggleFullscreen}
				class="px-4 py-2 bg-purple-600 hover:bg-purple-700 rounded-lg font-semibold transition-colors flex items-center gap-2"
			>
				{#if isFullscreen}
					<svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
						<path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="2"
							d="M6 18L18 6M6 6l12 12"
						/>
					</svg>
					Exit Fullscreen
				{:else}
					<svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
						<path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="2"
							d="M4 8V4h4M20 8V4h-4M4 16v4h4m12-4v4h4"
						/>
					</svg>
					Fullscreen
				{/if}
			</button>
		</div>

		<!-- Error Message -->
		{#if errorMessage}
			<div class="bg-red-500 bg-opacity-20 border border-red-500 rounded-lg p-4 mb-6">
				<p class="text-red-200">{errorMessage}</p>
			</div>
		{/if}

		<!-- Main Content Grid -->
		<div class="grid lg:grid-cols-3 gap-6">
			<!-- Video Section -->
			<div class="lg:col-span-2">
				<!-- Video Container -->
				<div class="relative bg-black rounded-lg overflow-hidden aspect-video mb-4 group">
					{#if hasPermission}
						<video
							bind:this={videoElement}
							class="w-full h-full transition-all duration-300"
							class:object-cover={videoFit === 'cover'}
							class:object-contain={videoFit === 'contain'}
							autoplay
							muted={false}
							playsinline
							style="transform: scaleX(-1); object-fit: {videoFit};"
						></video>

						<!-- Video controls overlay at bottom right -->
						<div
							class="absolute bottom-4 right-4 flex flex-col gap-2 opacity-0 group-hover:opacity-100 transition-opacity"
						>
							<!-- Aspect ratio toggle button -->
							<button
								onclick={() => (videoFit = videoFit === 'cover' ? 'contain' : 'cover')}
								class="p-2 bg-black bg-opacity-50 hover:bg-opacity-70 rounded-lg transition-all"
								title={videoFit === 'cover' ? 'Maintain aspect ratio' : 'Fill screen'}
							>
								{#if videoFit === 'cover'}
									<!-- Fit to screen icon (maintain aspect ratio) -->
									<svg
										class="w-5 h-5 text-white"
										fill="none"
										stroke="currentColor"
										viewBox="0 0 24 24"
									>
										<rect x="4" y="6" width="16" height="12" rx="1" stroke-width="2" />
										<path
											stroke-linecap="round"
											stroke-linejoin="round"
											stroke-width="2"
											d="M8 10h8M8 14h8"
										/>
									</svg>
								{:else}
									<!-- Fill screen icon (crop to fill) -->
									<svg
										class="w-5 h-5 text-white"
										fill="none"
										stroke="currentColor"
										viewBox="0 0 24 24"
									>
										<rect x="3" y="3" width="18" height="18" stroke-width="2" />
										<path
											stroke-linecap="round"
											stroke-linejoin="round"
											stroke-width="2"
											d="M3 8h18M3 16h18M8 3v18M16 3v18"
										/>
									</svg>
								{/if}
							</button>

							<!-- Fullscreen button -->
							<button
								onclick={toggleFullscreen}
								class="p-2 bg-black bg-opacity-50 hover:bg-opacity-70 rounded-lg transition-all"
								title="Toggle video fullscreen"
							>
								{#if isFullscreen}
									<!-- Exit fullscreen icon -->
									<svg
										class="w-5 h-5 text-white"
										fill="none"
										stroke="currentColor"
										viewBox="0 0 24 24"
									>
										<path
											stroke-linecap="round"
											stroke-linejoin="round"
											stroke-width="2"
											d="M6 18L18 6M6 6l12 12"
										/>
									</svg>
								{:else}
									<!-- Enter fullscreen icon -->
									<svg
										class="w-5 h-5 text-white"
										fill="none"
										stroke="currentColor"
										viewBox="0 0 24 24"
									>
										<path
											stroke-linecap="round"
											stroke-linejoin="round"
											stroke-width="2"
											d="M4 8V4h4M20 8V4h-4M4 16v4h4m12-4v4h4"
										/>
									</svg>
								{/if}
							</button>
						</div>

						<!-- Aspect ratio indicator -->
						<div
							class="absolute top-4 left-4 px-2 py-1 bg-black bg-opacity-50 rounded text-xs text-white opacity-0 group-hover:opacity-100 transition-opacity"
						>
							{videoFit === 'cover' ? 'Fill' : 'Fit'} ({videoFit})
						</div>
					{:else}
						<div class="flex items-center justify-center h-full">
							<div class="text-center">
								<div class="text-6xl mb-4">📹</div>
								<p class="text-gray-400">Camera and microphone access required</p>
							</div>
						</div>
					{/if}
				</div>

				<!-- Controls -->
				<div class="flex justify-center gap-4">
					{#if !hasPermission}
						<button
							onclick={requestPermissions}
							class="px-6 py-3 bg-blue-600 hover:bg-blue-700 rounded-lg font-semibold transition-colors"
						>
							Enable Camera & Microphone
						</button>
					{:else if !isStreaming}
						<button
							onclick={startStream}
							class="px-6 py-3 bg-green-600 hover:bg-green-700 rounded-lg font-semibold transition-colors"
						>
							Start Mirror
						</button>
					{:else}
						<button
							onclick={stopStream}
							class="px-6 py-3 bg-red-600 hover:bg-red-700 rounded-lg font-semibold transition-colors"
						>
							Stop Mirror
						</button>
					{/if}
				</div>
			</div>

			<!-- Audio Section -->
			<div class="space-y-4">
				<!-- Microphone Selection -->
				<div class="bg-gray-800 rounded-lg p-4">
					<h3 class="text-lg font-semibold mb-3">Microphone</h3>
					{#if availableMics.length > 0}
						<select
							bind:value={selectedMicId}
							onchange={switchMicrophone}
							class="w-full px-3 py-2 bg-gray-700 border border-gray-600 rounded-lg text-white focus:border-blue-500 focus:outline-none mb-3"
						>
							{#each availableMics as mic}
								<option value={mic.deviceId}>
									{mic.label || `Microphone ${mic.deviceId.slice(0, 8)}...`}
								</option>
							{/each}
						</select>

						<!-- Audio Delay Control -->
						<div class="mb-3">
							<div class="block text-sm text-gray-400 mb-2">
								Audio Delay: {audioDelay}ms
							</div>
							<input
								type="range"
								min="0"
								max="1000"
								step="10"
								bind:value={audioDelay}
								oninput={updateAudioDelay}
								class="w-full h-2 bg-gray-700 rounded-lg appearance-none cursor-pointer"
								disabled={!hasPermission || audioEnabled}
							/>
							<div class="flex justify-between text-xs text-gray-500 mt-1">
								<span>0ms</span>
								<span>500ms</span>
								<span>1000ms</span>
							</div>
						</div>

						<button
							onclick={toggleAudio}
							class="w-full px-3 py-2 rounded-lg font-semibold transition-colors flex items-center justify-center gap-2"
							class:bg-blue-600={!audioEnabled}
							class:bg-red-600={audioEnabled}
							class:hover:bg-blue-700={!audioEnabled}
							class:hover:bg-red-700={audioEnabled}
							disabled={!hasPermission}
						>
							{#if audioEnabled}
								<svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
									<path
										stroke-linecap="round"
										stroke-linejoin="round"
										stroke-width="2"
										d="M5.586 15H4a1 1 0 01-1-1v-4a1 1 0 011-1h1.586l4.707-4.707C10.923 3.663 12 4.109 12 5v14c0 .891-1.077 1.337-1.707.707L5.586 15z"
									/>
									<path
										stroke-linecap="round"
										stroke-linejoin="round"
										stroke-width="2"
										d="M17 14l2-2m0 0l2-2m-2 2l-2-2m2 2l2 2"
									/>
								</svg>
								Mute Audio Mirror
							{:else}
								<svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
									<path
										stroke-linecap="round"
										stroke-linejoin="round"
										stroke-width="2"
										d="M15.536 8.464a5 5 0 010 7.072m2.828-9.9a9 9 0 010 12.728M5.586 15H4a1 1 0 01-1-1v-4a1 1 0 011-1h1.586l4.707-4.707C10.923 3.663 12 4.109 12 5v14c0 .891-1.077 1.337-1.707.707L5.586 15z"
									/>
								</svg>
								Enable Audio Mirror
							{/if}
						</button>
					{:else}
						<p class="text-gray-400 text-sm">No microphones found</p>
					{/if}
				</div>

				<!-- Audio Level Meter -->
				<div class="bg-gray-800 rounded-lg p-4">
					<h3 class="text-lg font-semibold mb-3">Audio Level</h3>
					<div class="space-y-3">
						<!-- Simple Level Indicator -->
						<div class="w-full bg-gray-700 rounded-full h-4 overflow-hidden">
							<div
								class="h-full transition-all duration-100 rounded-full"
								class:bg-green-500={micLevel < 0.3}
								class:bg-yellow-500={micLevel >= 0.3 && micLevel < 0.7}
								class:bg-red-500={micLevel >= 0.7}
								style="width: {micLevel * 100}%"
							></div>
						</div>
						<p class="text-sm text-gray-400">Level: {Math.round(micLevel * 100)}%</p>

						<!-- Frequency Visualization -->
						<canvas
							bind:this={canvasElement}
							width="300"
							height="100"
							class="w-full h-24 bg-gray-700 rounded-lg"
						></canvas>
					</div>
				</div>

				<!-- Status -->
				<div class="bg-gray-800 rounded-lg p-4">
					<h3 class="text-lg font-semibold mb-3">Status</h3>
					<div class="space-y-2 text-sm">
						<div class="flex justify-between">
							<span class="text-gray-400">Streaming:</span>
							<span class:text-green-400={isStreaming} class:text-gray-500={!isStreaming}
								>{isStreaming ? 'Active' : 'Inactive'}</span
							>
						</div>
						<div class="flex justify-between">
							<span class="text-gray-400">Camera:</span>
							<span class:text-green-400={hasPermission} class:text-red-400={!hasPermission}
								>{hasPermission ? 'Connected' : 'Disconnected'}</span
							>
						</div>
						<div class="flex justify-between">
							<span class="text-gray-400">Microphone:</span>
							<span class:text-green-400={hasPermission} class:text-red-400={!hasPermission}
								>{hasPermission ? 'Connected' : 'Disconnected'}</span
							>
						</div>
						<div class="flex justify-between">
							<span class="text-gray-400">Audio Mirror:</span>
							<span class:text-green-400={audioEnabled} class:text-gray-500={!audioEnabled}
								>{audioEnabled ? 'Enabled' : 'Disabled'}</span
							>
						</div>
						<div class="flex justify-between">
							<span class="text-gray-400">Audio Delay:</span>
							<span class="text-gray-300">{audioDelay}ms</span>
						</div>
					</div>
				</div>
			</div>
		</div>

		<!-- Info Panel -->
		<div class="mt-8 bg-gray-800 rounded-lg p-6">
			<h2 class="text-xl font-semibold mb-4">How it works</h2>
			<ul class="space-y-2 text-gray-300">
				<li class="flex items-start">
					<span class="text-green-400 mr-2">•</span>
					<span>Click "Enable Camera & Microphone" to request device permissions</span>
				</li>
				<li class="flex items-start">
					<span class="text-green-400 mr-2">•</span>
					<span>Select your preferred microphone from the dropdown</span>
				</li>
				<li class="flex items-start">
					<span class="text-green-400 mr-2">•</span>
					<span>The video shows your camera feed in mirror mode (flipped horizontally)</span>
				</li>
				<li class="flex items-start">
					<span class="text-green-400 mr-2">•</span>
					<span>Audio from your microphone is played back through your speakers</span>
				</li>
				<li class="flex items-start">
					<span class="text-green-400 mr-2">•</span>
					<span>Hover over the video to reveal control buttons</span>
				</li>
				<li class="flex items-start">
					<span class="text-green-400 mr-2">•</span>
					<span><strong>Fit mode</strong>: Maintains aspect ratio with black borders</span>
				</li>
				<li class="flex items-start">
					<span class="text-green-400 mr-2">•</span>
					<span><strong>Fill mode</strong>: Crops video to fill the entire screen</span>
				</li>
				<li class="flex items-start">
					<span class="text-green-400 mr-2">•</span>
					<span>Use the fullscreen button for an immersive experience</span>
				</li>
				<li class="flex items-start">
					<span class="text-green-400 mr-2">•</span>
					<span>Adjust audio delay to synchronize lip movement with sound (0-1000ms)</span>
				</li>
			</ul>
		</div>
	</div>
</div>

<style>
	/* Ensure video maintains aspect ratio */
	video {
		width: 100%;
		height: 100%;
		object-fit: cover;
	}
</style>
