<script lang="ts">
	import { goto } from "$app/navigation";
	import { page } from "$app/state";
	import { PUBLIC_BACKEND_URL } from "$env/static/public";
	import {
		authState,
		Button,
		Field,
		Flex,
		Form,
		postFetch,
		TextField,
		toast,
		whenAuthReady
	} from "@davidnet-net/svelte-ui";

	let title = $state("");
	let files = $state<FileList | null>(null);
	let errorMessage = $state("");
	let isUploading = $state(false);

	// Ensure the user is logged in before allowing uploads
	$effect(() => {
		(async () => {
			await whenAuthReady();
			if (!authState.isLoggedIn && !authState.loading) {
				goto(`/login?continue=${encodeURIComponent(page.url.href)}`);
			}
		})();
	});

	async function handleSubmit(event: SubmitEvent) {
		event.preventDefault();

		if (!files || files.length === 0) {
			errorMessage = "Please select a video file to upload.";
			return;
		}

		errorMessage = "";
		isUploading = true;

		try {
			const formData = new FormData();
			formData.append("title", title);
			formData.append("video", files[0]);

			const result = await postFetch(
				`${PUBLIC_BACKEND_URL}/social/shorts`,
				formData,
				undefined,
				true
			);

			if (result.success) {
				toast(
					"Video Uploaded",
					"Your short has been published successfully.",
					"check_circle",
					4000,
					"success"
				);
				// Redirect to the newly uploaded video
				goto(`/shorts/${result.short.id}`);
			} else {
				errorMessage = result.error || "Failed to upload video.";
			}
		} catch (err) {
			console.error("Upload error:", err);
			errorMessage = "A network error occurred during upload.";
		} finally {
			isUploading = false;
		}
	}
</script>

<Flex justifyContent="center" alignItems="center" height="calc(100dvh - 56px)">
	<Form width="500px" onsubmit={handleSubmit}>
		<Field label="Video Title" name="title" required>
			{#snippet children()}
				<TextField
					bind:value={title}
					placeholder="Enter a descriptive title"
					maxlength={100}
					disabled={isUploading} />
			{/snippet}
		</Field>

		<Field label="Select Video" name="video" required invalid={errorMessage}>
			{#snippet children()}
				<input
					type="file"
					accept="video/mp4,video/webm,video/quicktime,video/x-matroska"
					bind:files
					required
					disabled={isUploading} />
			{/snippet}
		</Field>

		<Flex justifyContent="end">
			<Button type="submit" loading={isUploading} disabled={isUploading}>Upload video</Button>
		</Flex>
	</Form>
</Flex>
