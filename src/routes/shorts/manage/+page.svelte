<script lang="ts">
	import { goto } from "$app/navigation";
	import { Button, Field, Flex, Form, sleep, TextField } from "@davidnet-net/svelte-ui";
	let title = $state("");
	let files = $state<FileList | null>(null);
	let errorMessage = $state("");
	let isUploading = $state(false);

	async function handleSubmit(event: SubmitEvent) {
		event.preventDefault();

		if (!files || files.length === 0) {
			errorMessage = "Please select a video file to upload.";
			return;
		}

		errorMessage = "";
		isUploading = true;

		const formData = new FormData();
		formData.append("title", title);
		formData.append("video", files[0]);

		await sleep(2000);
		window.location.href = "https://home.davidnet.net/operation-teapot-nuke.gif";
		// Handle video upload request...
	}
</script>

<Flex justifyContent="center" alignItems="center">
	<Form width="500px" onsubmit={handleSubmit}>
		<Field label="Video Title" name="title" required>
			{#snippet children()}
				<TextField bind:value={title} placeholder="Enter a descriptive title" maxlength={100} />
			{/snippet}
		</Field>

		<Field label="Select Video" name="video" required invalid={errorMessage}>
			{#snippet children()}
				<input type="file" accept="video/*" bind:files required />
			{/snippet}
		</Field>

		<Flex justifyContent="end">
			<Button type="submit" disabled={isUploading}>
				{isUploading ? "Uploading..." : "Upload video"}
			</Button>
		</Flex>
	</Form>
</Flex>
