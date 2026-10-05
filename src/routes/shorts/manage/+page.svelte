<script lang="ts">
	import { goto } from "$app/navigation";
	import { page } from "$app/state";
	import { PUBLIC_BACKEND_URL } from "$env/static/public";
	import {
		Anchor,
		authState,
		Button,
		Field,
		Flex,
		Form,
		Link,
		postFetch,
		TextField,
		toast,
		whenAuthReady
	} from "@davidnet-net/svelte-ui";
	import * as m from "$lib/paraglide/messages.js";

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
			errorMessage = m.page_shorts_manage_error_no_file();
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
					m.page_shorts_manage_toast_success_title(),
					m.page_shorts_manage_toast_success_content(),
					"check_circle",
					4000,
					"success"
				);
				// Redirect to the newly uploaded video
				goto(`/shorts/${result.short.id}`);
			} else {
				errorMessage = result.error || m.page_shorts_manage_error_upload_failed();
			}
		} catch (err) {
			console.error("Upload error:", err);
			errorMessage = m.page_shorts_manage_error_network();
		} finally {
			isUploading = false;
		}
	}
</script>

<Flex justifyContent="center" alignItems="center" height="calc(100dvh - 56px)">
	<Form width="500px" onsubmit={handleSubmit}>
		<Field label={m.page_shorts_manage_upload_title_label()} name="title" required>
			{#snippet children()}
				<TextField
					bind:value={title}
					placeholder={m.page_shorts_manage_upload_title_placeholder()}
					maxlength={100}
					disabled={isUploading} />
			{/snippet}
		</Field>

		<p>
			{m.page_shorts_manage_legal_prefix()}<Link
				opennewtab
				href="https://davidnet.net/legal/acceptable_use_policy">
				{m.page_shorts_manage_legal_aup()}
			</Link>{m.page_shorts_manage_legal_and()}<Link
				opennewtab
				href="https://davidnet.net/legal/community_guidelines">
				{m.page_shorts_manage_legal_guidelines()}
			</Link>{m.page_shorts_manage_legal_suffix()}
		</p>
		<Field label={m.page_shorts_manage_select_video_label()} name="video" required invalid={errorMessage}>
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
			<Button type="submit" loading={isUploading} disabled={isUploading}
				>{m.page_shorts_manage_submit()}</Button>
		</Flex>
	</Form>
</Flex>
