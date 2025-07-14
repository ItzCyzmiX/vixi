<script>
	import Groq, { toFile } from 'groq-sdk';
	import { onMount } from 'svelte';
	import { fly, scale, slide } from 'svelte/transition';
	
	const groq = new Groq({
		apiKey: import.meta.env.VITE_GROQ_KEY,
		dangerouslyAllowBrowser: true
	});

	let isPaused = $state(false);
	let isAiSpeaking = $state(false);
	let callStartTime = $state(Date.now());
	let elapsedTime = $state(0);
	let callEnded = $state(false);
	let mediaRecorder = $state(null);
	let chunks = $state([]);
	let callTimer;
	let speakingTimer;
	let callStarted = $state(false);
	let chatHistory = $state([]);
	let callState = $state('Listening...');
	let mode = $state('call'); // 'call' or 'chat'
	let chatInput = $state('');
	let chatLoading = $state(false);
	let selectedLanguage = $state('en-US');
	const languages = [
		{ code: 'en-US', label: 'English' },
		{ code: 'es-ES', label: 'Español' },
		{ code: 'fr-FR', label: 'Français' }
	];
	// Format time display
	const formatTime = (seconds) => {
		const minutes = Math.floor(seconds / 60);
		const secs = seconds % 60;
		return `${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
	};

	function isAlphanumeric(char) {
		if (char === " ") return true
		
  	return /^[a-zA-Z0-9]$/.test(char);
	}
	
	$effect(() => {
		if (isPaused || mediaRecorder?.state === 'recording') {
			mediaRecorder?.pause();
		}
		if (!isPaused && mediaRecorder?.state === 'paused') {
			mediaRecorder?.resume();
		}
	});

	const pauseCall = () => {
		isPaused = !isPaused;
		if (isPaused) {
			callState = 'Paused';
			mediaRecorder?.pause();
			clearInterval(callTimer);
			clearInterval(speakingTimer);
		} else {
			callState = 'Listening...';
			mediaRecorder?.resume();
			callStartTime = Date.now() - elapsedTime * 1000; // Adjust start time
			callTimer = setInterval(() => {
				elapsedTime = Math.floor((Date.now() - callStartTime) / 1000);
			}, 1000);
		}
	};
	// End call
	const endCall = () => {
		callEnded = true;
		isAiSpeaking = false;
		mediaRecorder?.stop();
		clearInterval(callTimer);
		clearInterval(speakingTimer);
	};
	async function speechToText(audio) {
		if (callEnded) {
			return;
		}
		callState = 'Transcribing...';
		const transcription = await groq.audio.transcriptions.create({
			file: audio,
			model: 'whisper-large-v3-turbo',
			response_format: 'verbose_json',
			language: selectedLanguage.split('-')[0] // Use language code from selectedLanguage
		});
		callState = 'Thinking...';
		let aiMessage = '';
		let res = await groq.chat.completions.create({
			messages: [
				{
					role: 'system',
					content:
						'You are a helpfull ai assitant, your name is vixi'
				},
				...chatHistory,
				{
					role: 'user',
					content: transcription.text
				}
			],

			model: 'llama-3.3-70b-versatile'
		});
		aiMessage = res.choices[0].message.content;
		const chatMessage = [
			{
				role: 'user',
				content: transcription.text
			},
			{
				role: 'assistant',
				content: aiMessage || ''
			}
		];
		chatHistory = [...chatHistory, ...chatMessage]; // Update chat history
		callState = 'Speaking...';
		console.log(aiMessage)
		let messageWithoutAnnotations = aiMessage.split('*');
		messageWithoutAnnotations = messageWithoutAnnotations
			.map((msg, i) => {
				if ((i + 1) % 2 == 0) {
					return '';
				}

				

				return msg.trim();
			})
			.join(' ')
		puter.ai
			.txt2speech(messageWithoutAnnotations || '', selectedLanguage)
			.then((audio) => {
				console.log(audio);
				isAiSpeaking = true; // Start animation
				audio.play();
				callState = '';
				audio.onended = (e) => {
					isAiSpeaking = false; // Stop animation when audio finishes
					callState = 'Listening...';
					setTimeout(() => {
						mediaRecorder?.start();
					}, 700);
				};
			});
	}
	function startCall() {
		callStarted = true;
		callEnded = false;
		callStartTime = Date.now();
		elapsedTime = 0;

		// Start call timer
		callTimer = setInterval(() => {
			elapsedTime = Math.floor((Date.now() - callStartTime) / 1000);
		}, 1000);

		if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
			navigator.mediaDevices
				.getUserMedia({ audio: true })
				.then((stream) => {
					mediaRecorder = new MediaRecorder(stream);
					mediaRecorder.start();
					let silenceTimeout;
					let audioContext = new (window.AudioContext || window.webkitAudioContext)();
					let source = audioContext.createMediaStreamSource(stream);
					let analyser = audioContext.createAnalyser();
					source.connect(analyser);
					analyser.fftSize = 2048;
					let dataArray = new Uint8Array(analyser.fftSize);

					let silenceLevel = 0.05; // Increase this for noisy environments (try 0.03–0.05)
					let silenceDuration = 3000; // ms of silence before stopping (increase if needed)

					function checkSilence() {
						analyser.getByteTimeDomainData(dataArray);
						// Calculate RMS (root mean square) for volume
						let sum = 0;
						for (let i = 0; i < dataArray.length; i++) {
							let val = (dataArray[i] - 128) / 128;
							sum += val * val;
						}
						let rms = Math.sqrt(sum / dataArray.length);

						// Optionally, use a moving average to smooth out spikes
						if (!checkSilence.rmsHistory) checkSilence.rmsHistory = [];
						const rmsHistory = checkSilence.rmsHistory;
						rmsHistory.push(rms);
						if (rmsHistory.length > 10) rmsHistory.shift(); // last 10 frames
						const avgRms = rmsHistory.reduce((a, b) => a + b, 0) / rmsHistory.length;

						if (avgRms < silenceLevel) {
							if (!silenceTimeout) {
								silenceTimeout = setTimeout(() => {
									console.log('User has been quiet for ~3 seconds');
									if (mediaRecorder.state === 'recording') {
										mediaRecorder.stop();
									}
								}, silenceDuration);
							}
						} else {
							if (silenceTimeout) {
								clearTimeout(silenceTimeout);
								silenceTimeout = null;
							}
						}
						if (!callEnded) {
							requestAnimationFrame(checkSilence);
						}
					}
					checkSilence();

					mediaRecorder.ondataavailable = (e) => {
						chunks = [...chunks, e.data];
					};
					mediaRecorder.onstop = async () => {
						const audioBlob = new Blob(chunks, { type: 'audio/webm' });
						chunks = []; // Clear chunks after saving
						const file = await toFile(audioBlob, 'audio.webm');
						await speechToText(file);
					};
				})
				.catch((err) => {
					console.error(`The following getUserMedia error occurred: ${err}`);
				});
		} else {
			console.log('getUserMedia not supported on your browser!');
		}
	}

	onMount(async () => {
		// Initialize call state
		callStarted = false;
		isAiSpeaking = false;
		callEnded = false;
		elapsedTime = 0;

		return () => {
			clearInterval(callTimer);
			clearInterval(speakingTimer);
		};
	});
	async function sendChat(event) {
		event.preventDefault();
		if (!chatInput.trim()) return;

		chatLoading = true;

		const res = await groq.chat.completions.create({
			messages: [
				{
					role: 'system',
					content:
						'You are a helpfull ai assitant, your name is vixi'
				},
				...chatHistory,
				{
					role: 'user',
					content: chatInput
				}
			],
			model: 'llama-3.3-70b-versatile'
		});
		const aiMessage = res.choices[0].message;

		const aiMsg = [
			{ role: 'user', content: chatInput },
			{
				role: 'assistant',
				content: aiMessage.content || ''
			}
		];
		chatHistory = [...chatHistory, ...aiMsg];
		chatInput = '';
		chatLoading = false;
	}
</script>

<!-- Mode Toggle -->
<div class="absolute top-4 right-4 flex gap-2 z-10">
	<button
		class="px-4 py-2 rounded-lg text-sm font-semibold transition-all duration-200"
		class:bg-blue-600={mode === 'call'}
		class:bg-gray-700={mode !== 'call'}
		class:text-white={mode === 'call'}
		class:text-gray-300={mode !== 'call'}
		onclick={() => (mode = 'call')}
	>
		Call Mode
	</button>
	<button
		class="px-4 py-2 rounded-lg text-sm font-semibold transition-all duration-200"
		class:bg-blue-600={mode === 'chat'}
		class:bg-gray-700={mode !== 'chat'}
		class:text-white={mode === 'chat'}
		class:text-gray-300={mode !== 'chat'}
		onclick={() => (mode = 'chat')}
	>
		Chat Mode
	</button>
</div>
<div class="absolute top-4 left-4 z-10">
	<select
		class="bg-blue-700 border border-gray-700 text-gray-100 px-10 py-2 rounded-lg focus:outline-none focus:border-blue-500 transition"
		bind:value={selectedLanguage}
	>
		{#each languages as lang}
			<option value={lang.code}>{lang.label}</option>
		{/each}
	</select>
</div>
{#if mode === 'call'}
	<!-- ...existing call UI... -->
	{#if !callStarted}
		<div class="min-h-screen bg-black flex items-center justify-center p-4">
			<button
				class="px-8 py-4 rounded-2xl bg-blue-600 text-white text-xl font-semibold shadow-lg hover:bg-blue-700 transition"
				onclick={startCall}
			>
				Start Call
			</button>
		</div>
	{:else}
		<div class="min-h-screen bg-black flex items-center justify-center p-4">
			<div
				class="w-98 h-[720px] bg-neutral-950 rounded-3xl border border-white/30 flex flex-col items-center p-8 relative shadow-lg shadow-gray-800/50"
				transition:fly={{ y: 100, duration: 300 }}
			>
				<!-- Time Display -->
				<div class="absolute top-10 text-gray-500 text-base font-light">
					{formatTime(elapsedTime)}
				</div>

				<!-- AI Name and Status -->
				<div class="text-white text-2xl font-normal mb-2 mt-10">VIXI</div>

				<!-- Call State Indicator -->
				{#if !callEnded}
					<div class="flex items-center gap-2 mb-10">
						{#if callState === 'Listening...'}
							<span class="status-dot bg-green-400 animate-pulse"></span>
							<span class="text-green-300 font-medium">{callState}</span>
						{:else if callState === 'Transcribing...'}
							<span class="status-dot bg-yellow-400 animate-spin"></span>
							<span class="text-yellow-300 font-medium">{callState}</span>
						{:else if callState === 'Thinking...'}
							<span class="status-dot bg-blue-400 animate-bounce"></span>
							<span class="text-blue-300 font-medium">{callState}</span>
						{:else if callState === 'Speaking...'}
							<span class="status-dot bg-purple-400 animate-pulse"></span>
							<span class="text-purple-300 font-medium">{callState}</span>
						{:else}
							{callState}
						{/if}
					</div>
				{/if}

				<!-- AI Avatar -->
				<!-- svelte-ignore a11y_no_noninteractive_tabindex -->
				<div
					class="w-40 h-40 rounded-full mt-10 relative overflow-hidden cursor-pointer transition-all duration-300 ease-in-out"
					class:speaking={isAiSpeaking}
					tabindex="0"
				>
					<!-- Base gradient background -->
					<div
						class="absolute inset-0 rounded-full bg-gradient-to-br from-gray-800 via-gray-700 to-gray-600 animate-gradient-shift"
					></div>

					<!-- Color overlays with backdrop blur -->
					<div
						class="absolute inset-0 rounded-full animate-color-shift opacity-30 backdrop-blur-md"
					>
						<div
							class="absolute inset-0 bg-gradient-radial from-blue-400/30 via-transparent to-transparent"
						></div>
						<div
							class="absolute inset-0 bg-gradient-radial from-pink-400/30 via-transparent to-transparent transform rotate-120"
						></div>
						<div
							class="absolute inset-0 bg-gradient-radial from-green-400/20 via-transparent to-transparent transform rotate-240"
						></div>
					</div>
				</div>

				<!-- Controls -->
				<div class="flex gap-10 mt-auto">
					<!-- End Call Button -->
					<button
						class="w-14 h-14 rounded-full border border-gray-700 bg-gray-800 text-white flex items-center justify-center text-lg transition-all duration-300 hover:bg-gray-700 disabled:opacity-40"
						onclick={endCall}
						disabled={callEnded}
					>
						✕
					</button>
					<button
						class="w-14 h-14 rounded-full border border-gray-700 bg-gray-800 text-white flex items-center justify-center text-lg transition-all duration-300 hover:bg-gray-700 disabled:opacity-40"
						onclick={pauseCall}
						disabled={callEnded}
					>
						PAUSE
					</button>
				</div>
			</div>
		</div>
	{/if}
{:else}
	<!-- Chat Mode UI -->
	<div class="min-h-screen bg-black flex items-center justify-center p-4">
		<div
			class="w-114 h-[800px] bg-neutral-950 rounded-3xl border border-white/30 flex flex-col items-center p-8 relative shadow-lg shadow-gray-800/50"
			transition:slide={{ duration: 200, axis: 'y' }}
		>
			<div class="absolute top-10 text-gray-500 text-base font-light">Chat Mode</div>
			<div class="text-white text-2xl font-normal mb-2 mt-10">Claude</div>
			<div class="text-gray-500 text-sm mb-2">Connected</div>
			<div class="flex-1 w-full overflow-y-auto mb-6 mt-4 px-2">
				{#each chatHistory as msg}
					<div class="mb-4 flex {msg.role === 'user' ? 'justify-end' : 'justify-start'}">
						<div class="flex flex-col items-{msg.role === 'user' ? 'end' : 'start'} max-w-[80%]">
							<span class="block text-xs text-gray-500 mb-1 select-none">
								{msg.role === 'user' ? 'You' : 'Claude'}
							</span>
							<div
								class="rounded-2xl px-5 py-3 shadow-md transition-all duration-200"
								class:bg-gray-800={msg.role === 'user'}
								class:bg-gray-900={msg.role !== 'user'}
								class:border={msg.role !== 'user'}
								class:border-gray-700={msg.role !== 'user'}
								class:text-gray-100={msg.role !== 'user'}
								class:text-gray-300={msg.role === 'user'}
								class:text-right={msg.role === 'user'}
								class:text-left={msg.role !== 'user'}
								style="word-break: break-word;"
							>
								{msg.content}
							</div>
						</div>
					</div>
				{/each}
				{#if chatLoading}
					<div class="text-gray-400 italic text-center mt-2">Claude is typing...</div>
				{/if}
			</div>
			<form class="w-full flex gap-2 mt-auto" onsubmit={sendChat} autocomplete="off">
				<input
					class="flex-1 px-4 py-3 rounded-xl bg-gray-900 text-gray-100 border border-gray-700 focus:outline-none focus:border-blue-500 transition placeholder-gray-500"
					bind:value={chatInput}
					placeholder="Type your message..."
					disabled={chatLoading}
					autocomplete="off"
				/>
				<button
					class="px-6 py-3 rounded-xl bg-blue-600 text-white font-semibold hover:bg-blue-700 transition disabled:opacity-60"
					type="submit"
					disabled={chatLoading}
				>
					Send
				</button>
			</form>
		</div>
	</div>
{/if}

<style>
	.speaking {
		@apply scale-105;
		box-shadow: 0 0 40px rgba(100, 200, 255, 0.3);
		animation: pulse 1.5s ease-in-out infinite;
	}

	.speaking .animate-color-shift {
		@apply opacity-60;
		animation:
			color-shift 1.5s ease-in-out infinite,
			color-pulse 2s ease-in-out infinite;
	}

	@keyframes gradient-shift {
		0% {
			background-position: 0% 50%;
		}
		50% {
			background-position: 100% 50%;
		}
		100% {
			background-position: 0% 50%;
		}
	}

	@keyframes color-shift {
		0% {
			transform: rotate(0deg) scale(1);
		}
		33% {
			transform: rotate(120deg) scale(1.1);
		}
		66% {
			transform: rotate(240deg) scale(0.9);
		}
		100% {
			transform: rotate(360deg) scale(1);
		}
	}

	@keyframes pulse {
		0%,
		100% {
			transform: scale(1.05);
		}
		50% {
			transform: scale(1.1);
		}
	}

	@keyframes color-pulse {
		0%,
		100% {
			opacity: 0.6;
		}
		50% {
			opacity: 0.4;
		}
	}

	.animate-gradient-shift {
		background-size: 300% 300%;
		animation: gradient-shift 8s ease-in-out infinite;
	}

	.animate-color-shift {
		animation: color-shift 6s ease-in-out infinite;
	}

	.bg-gradient-radial {
		background: radial-gradient(circle at center, var(--tw-gradient-stops));
	}

	.transform.rotate-120 {
		transform: rotate(120deg);
	}

	.transform.rotate-240 {
		transform: rotate(240deg);
	}
	.status-dot {
		width: 0.75rem;
		height: 0.75rem;
		border-radius: 9999px;
		display: inline-block;
	}
	@keyframes spin {
		0% {
			transform: rotate(0deg);
		}
		100% {
			transform: rotate(360deg);
		}
	}
	.animate-spin {
		animation: spin 1s linear infinite;
	}
	input::placeholder {
		color: #6b7280; /* Tailwind's gray-500 */
		opacity: 1;
	}
	form input:disabled {
		background-color: #23272e;
		color: #6b7280;
	}
	form button:disabled {
		cursor: not-allowed;
	}
</style>
