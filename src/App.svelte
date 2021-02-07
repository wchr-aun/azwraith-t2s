<script>
	import { Stretch } from 'svelte-loading-spinners'
	import { Button } from 'svelma'

	let textDump = 'a';
	let textarea;
	let loading = false;

	function enterPressEvent(event) {
		if (document.activeElement !== textarea && event.key === 'Enter') {
			text2speech();
		}
	}

	function text2speech() {
		if (!!textDump) {
			const message = new SpeechSynthesisUtterance(textDump);
			loading = true;
			new Promise((resolve, reject) => {
				const voices = window.speechSynthesis;
				setInterval(() => {
					if (voices.getVoices().length !== 0) {
						resolve(voices.getVoices()[4]);
						clearInterval();
					}
				}, 10)
			}).then(voice => {
				setTimeout(() => {
					loading = false;
					message.voice = voice;
					message.rate = 0.85;
					window.speechSynthesis.speak(message);
				}, 500)
			});
		}
	}

	function pause() {
		window.speechSynthesis.pause();
	}
</script>

<svelte:window on:keydown={enterPressEvent}/>
<div class="container middle" on:keypress={enterPressEvent}>
	<div class="{loading === false ? 'hide' : ''}">
		<div class="overlay is-flex is-justify-content-center is-align-items-center">
			<p></p><Stretch color="#666666" unit="px" duration="1s"></Stretch>
		</div>
	</div>
	<span class="is-uppercase is-size-3">Text Dumpping</span>
	<textarea bind:this={textarea} class="textDump" bind:value={textDump}></textarea>
	<Button class="is-pulled-right block is-info" on:click={text2speech}>Speak</Button>
</div>

<style>
	.textDump {
		width: 100%;
		height: 50vh;
	}
	.middle {
		margin: auto;
		width: 50%;
		padding: 10px;
	}
	.overlay {
		position: fixed;
		display: none;
		width: 100%;
		height: 100%;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background-color: rgba(0,0,0,0.8);
		z-index: 2;
		cursor: not-allowed;
	}
	.hide {
		display: none;
	}
</style>