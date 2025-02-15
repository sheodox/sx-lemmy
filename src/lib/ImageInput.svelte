<style lang="scss">
	.image-input {
		border: 1px solid var(--sx-gray-transparent-lighter);
		overflow: hidden;
	}
	.input-container.has-upload {
		border-bottom: 1px solid var(--sx-gray-transparent-lighter);
	}
	label {
		border-right: 1px solid var(--sx-gray-transparent-lighter);
		background: var(--sx-gray-transparent-dark);
		line-height: 1.7;
	}
	.input-container {
		align-items: stretch;
	}
	.input-busy {
		width: 0;
		visibility: hidden;
	}
	.current-url-preview + .input-container {
		border-top: 1px solid var(--sx-gray-transparent-lighter);
	}
</style>

<div class="image-input f-column card card-body p-0">
	{#if name}
		<input type="hidden" value={currentUploadUrl} {name} />
	{/if}
	{#if showCurrentUpload && currentUploadUrl}
		<div class="current-url-preview p-4">
			<UploadedMedia bind:currentUploadUrl />
		</div>
	{/if}
	<div class="input-container f-row gap-4" class:has-upload={!!mostRecentUpload}>
		<label for={fileInputId} class="p-4"> <Icon icon="image" /> {label} </label>
		<input
			type="file"
			accept="image/*,video/*"
			id={fileInputId}
			class="py-4"
			on:change={onFileInputChange}
			class:px-4={!busy}
			class:input-busy={busy}
			disabled={busy}
			bind:this={elFileInput}
		/>
		{#if busy}
			<div class="f-row justify-content-center align-items-center">
				<Spinner />
			</div>
		{/if}
	</div>
	{#if mostRecentUpload}
		<div class="m-4">
			{#key mostRecentUpload}
				<UploadedMedia upload={mostRecentUpload} on:delete-upload={onDeleteUpload} />
			{/key}
		</div>
	{/if}
</div>

<script lang="ts">
	import { Icon } from 'sheodox-ui';
	import type { UploadImageResponse } from 'lemmy-js-client';
	import { createEventDispatcher } from 'svelte';
	import { profile } from './profiles/profiles';
	import UploadedMedia from './UploadedMedia.svelte';
	import Spinner from './Spinner.svelte';
	import { genId } from 'sheodox-ui/util';

	export let label = 'Image';
	export let showCurrentUpload = false;
	export let currentUploadUrl = '';
	// 'name' field on a hidden input, value=currentUploadUrl
	export let name = '';

	const fileInputId = `post-compose-image-file-input-${genId()}`;

	const dispatch = createEventDispatcher<{
		// when an uploaded image is deleted
		'delete-upload': UploadImageResponse;
		// when an image is uplaoded
		upload: UploadImageResponse;
	}>();

	function onDeleteUpload(e: CustomEvent<UploadImageResponse>) {
		mostRecentUpload = null;
		currentUploadUrl = '';
		dispatch('delete-upload', e.detail);
	}

	// if currently doing an upload
	let busy = false;

	let mostRecentUpload: null | UploadImageResponse = null;

	let elFileInput: HTMLInputElement;

	$: client = $profile.client;

	async function onFileInputChange(e: Event & { currentTarget: EventTarget & HTMLInputElement }) {
		const file = e.currentTarget?.files?.item(0);
		if (file && (file.type.includes('image') || file.type.includes('video'))) {
			e.currentTarget.value = '';
			busy = true;

			try {
				const res = await client.uploadImage({
					image: file
				});

				mostRecentUpload = res;
				currentUploadUrl = res.url ?? '';
				dispatch('upload', res);
			} finally {
				busy = false;
			}
		}
	}
</script>
