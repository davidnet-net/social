<script lang="ts">
	import { postFetch, whenAuthReady } from "@davidnet-net/svelte-ui";
	import { onMount } from "svelte";

	onMount(async () => {
		await whenAuthReady();
		postFetch(
			"https://davidnet-backend.davidnet.net/social/shorts/admin/backfill-durations",
			{},
			undefined,
			true
		);
	});
</script>
