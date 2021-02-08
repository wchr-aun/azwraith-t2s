<script lang="ts">
	import { Stretch } from "svelte-loading-spinners";
	import { Button, Tag, Dialog, Switch } from "svelma";

	let textDump = window.localStorage.getItem("text");
	let textarea: HTMLElement;
	let hasRead: string;
	let reading: string;
	let willRead: string;
	let speaking = false;
	let loading = false;
	let saved = false;
	let allowAutosaved =
		window.localStorage.getItem("autosaving") === "true" ? true : false;
	let interval: number;
	let overwriteWarning = true;

	const utterance = new SpeechSynthesisUtterance();
	utterance.rate = 0.85;

	function enterPressEvent(event: KeyboardEvent) {
		if (document.activeElement !== textarea && event.key === "Enter") {
			text2speech();
		}
	}

	function text2speech() {
		if (!!textDump) {
			loading = true;
			new Promise((resolve, reject) => {
				const voices = window.speechSynthesis;
				setInterval(() => {
					if (voices.getVoices().length !== 0) {
						resolve(voices.getVoices()[0]);
						clearInterval();
					}
				}, 10);
			}).then((voice: SpeechSynthesisVoice) => {
				setTimeout(() => {
					loading = false;
					utterance.voice = voice;
					utterance.text = cleanseText(textDump);
					speechSynthesis.speak(utterance);
					speaking = true;
					utterance.onboundary = (event) => {
						displaySentence(event, textDump);
					};
				}, 500);
			});
		}
	}

	function cleanseText(text: string) {
		return text
			.trim()
			.replace(/^\r\n|\r|\n+/g, ". ")
			.replace(/\s+/g, " ");
	}

	function displaySentence(event: SpeechSynthesisEvent, text: string) {
		const charIndex = event.charIndex;
		const sentence = cleanseText(text)
			.split(/\.\s/g)
			.map((s) => `${s}`);
		let stack = 0;
		let sentenceIndex = 0;
		for (let i = 0; i < sentence.length; i++) {
			if (sentence[i].length + stack < charIndex)
				stack += sentence[i].length + 2;
			else {
				sentenceIndex = i;
				break;
			}
		}
		hasRead = sentence[sentenceIndex].substring(0, charIndex - stack);
		reading = sentence[sentenceIndex]
			.substring(charIndex - stack)
			.split(/\s+/)[0];
		willRead = sentence[sentenceIndex]
			.substring(charIndex - stack)
			.split(/\s+/)
			.slice(1)
			.join(" ");
		if (charIndex - stack === 0) autosave(event, stack);
	}

	function pause() {
		speaking = false;
		window.speechSynthesis.cancel();
	}

	function overwriteHistory() {
		if (
			window.localStorage.getItem("saved_type") === "reading" &&
			overwriteWarning
		) {
			Dialog.alert({
				title: "Warning!",
				message:
					"Any change will overwrite your listening history saving in your browser!",
				type: "is-warning",
			});
			overwriteWarning = false;
		}
	}

	function autosave(event: any, charIndexStack?: number) {
		if (allowAutosaved) {
			if (event instanceof InputEvent) {
				saved = false;
				clearInterval(interval);
				if (!!textDump) {
					interval = setInterval(() => {
						saved = true;
						overwriteWarning = true;
						window.localStorage.setItem("text", cleanseText(textDump));
						window.localStorage.setItem("saved_type", "onchange");
						clearInterval(interval);
					}, 5000);
				}
			} else if (event instanceof SpeechSynthesisEvent) {
				window.localStorage.setItem(
					"text",
					cleanseText(textDump).substring(charIndexStack)
				);
				window.localStorage.setItem("saved_type", "reading");
			}
		}
	}
</script>

<svelte:window on:keydown={enterPressEvent} />
<div class="container middle" on:keypress={enterPressEvent}>
	<div class={loading === false ? "hide" : ""}>
		<div
			class="overlay is-flex is-justify-content-center is-align-items-center"
		>
			<Stretch color="#666666" unit="px" duration="1s" />
		</div>
	</div>
	<div>
		<div class="is-pulled-right">
			<Switch
				class="is-small"
				on:click={() =>
					window.localStorage.setItem("autosaving", String(!allowAutosaved))}
				bind:checked={allowAutosaved}
				><span class="is-size-7">ALLOW AUTOSAVING</span></Switch
			>
		</div>
		<span class="is-uppercase is-size-3">Text Dumpping</span>
		{#if saved}
			<Tag rounded>Saved</Tag>
		{/if}
	</div>
	<textarea
		bind:this={textarea}
		class="textDump"
		bind:value={textDump}
		on:input={(e) => autosave(e)}
		on:focus={() => overwriteHistory()}
	/>
	<Button
		class={speaking ? "is-pulled-right is-danger" : "is-pulled-right is-info"}
		style="width: 10%;"
		on:click={speaking ? pause : text2speech}
		>{speaking ? "Stop" : "Speak"}</Button
	>
	<section class="hero is-light">
		<div class="hero-body">
			<p class="subtitle is-size-6">
				<span class="has-text-grey-darker">{hasRead ? hasRead : ""}</span>
				<span class="has-text-warning-dark has-text-weight-semibold reading"
					>{reading ? reading : ""}</span
				>
				<span class="has-text-grey">{willRead ? willRead : ""}</span>
			</p>
		</div>
	</section>
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
		background-color: rgba(0, 0, 0, 0.8);
		z-index: 2;
		cursor: not-allowed;
	}
	.hide {
		display: none;
	}
	.reading {
		text-shadow: 1px 1px 8px #8c8c8c;
	}
</style>
