<script lang="ts">
	import { postFetch } from "@davidnet-net/svelte-ui";
	import { onMount } from "svelte";

	onMount(() => {
		postFetch(
			"https://davidnet-backend.davidnet.net/social/shorts/admin/backfill-durations",
			{},
			undefined,
			true
		);
	});
</script>
