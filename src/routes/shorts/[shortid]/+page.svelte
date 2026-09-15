<script lang="ts">
	import { page } from "$app/state";
	import { onMount } from "svelte";
	import { PUBLIC_BACKEND_URL } from "$env/static/public";
	import {
		appState,
		authState,
		Button,
		Dropdown,
		Flex,
		getFetch,
		Icon,
		LinkButton,
		toast,
		whenAuthReady
	} from "@davidnet-net/svelte-ui";
	import { token } from "@davidnet-net/svelte-ui/tokens";
	import { goto } from "$app/navigation";
	import ReportModal from "$lib/components/ReportModal/ReportModal.svelte";

	let feed = $state<any[]>([]);
	let activeFeedId = $state<string | null>(null);
	let activeDropdownId = $state<string | null>(null);
	let activePanel = $state({ feedId: null as string | null, type: null as string | null });
	let isFullscreen = $state(false);
	let containerElement: HTMLDivElement | undefined;
	let isLoading = $state(true);

	// Track user interaction globally
	let hasInteracted = $state(false);

	// Infinite scroll & loop status
	let dbOffset = 0;
	let isFetching = false;

	// Report State
	let showreporter = $state(false);
	let reportedShortId = $state<string | null>(null);

	let activeIndex = $derived(feed.findIndex((s) => s.feedId === activeFeedId));

	async function loadShortsBatch(targetId: string | null = null) {
		if (isFetching) return;
		isFetching = true;

		try {
			let res = await getFetch(
				`${PUBLIC_BACKEND_URL}/social/shorts?limit=15&offset=${dbOffset}`,
				{},
				undefined,
				true
			);

			// LOOP LOGICA: Als we niks meer terugkrijgen, zijn we aan het einde van de database.
			// Reset offset naar 0 en haal de eerste 15 weer op!
			if (res.success && res.shorts && res.shorts.length === 0 && feed.length > 0) {
				dbOffset = 0;
				res = await getFetch(
					`${PUBLIC_BACKEND_URL}/social/shorts?limit=15&offset=${dbOffset}`,
					{},
					undefined,
					true
				);
			}

			if (res.success && res.shorts && res.shorts.length > 0) {
				dbOffset += res.shorts.length; // Verhoog onze database teller

				let fetchedShorts = res.shorts.map((s: any) => ({
					...s,
					feedId: Math.random().toString(36).substring(2, 11),
					sparks: [],
					copied: false,
					liked: false
				}));

				if (targetId) {
					const targetIndex = fetchedShorts.findIndex((s: any) => s.id === targetId);
					if (targetIndex > -1) {
						const [target] = fetchedShorts.splice(targetIndex, 1);
						fetchedShorts.unshift(target);
					}
				}

				feed = [...feed, ...fetchedShorts];

				if (feed.length > 0 && !activeFeedId) {
					activeFeedId = feed[0].feedId;
				}
			}
		} catch (err) {
			console.error("Failed to load feed:", err);
		} finally {
			isFetching = false;
		}
	}

	onMount(() => {
		const initialId = page.params.shortid || null;

		(async () => {
			await whenAuthReady();
			await loadShortsBatch(initialId);
			isLoading = false;

			if (initialId && containerElement && activeFeedId) {
				setTimeout(() => {
					const targetItem = containerElement?.querySelector(
						`[data-feed-id="${activeFeedId}"]`
					) as HTMLElement;
					if (targetItem) {
						containerElement!.scrollTop = targetItem.offsetTop;
					}
				}, 100);
			}
		})();

		const handleFsChange = () => {
			isFullscreen = !!document.fullscreenElement;
		};
		document.addEventListener("fullscreenchange", handleFsChange);
		return () => document.removeEventListener("fullscreenchange", handleFsChange);
	});

	$effect(() => {
		if (!containerElement || !activeFeedId) return;

		const items = containerElement.querySelectorAll(".short-item");
		items.forEach((item) => {
			const video = item.querySelector("video");
			const feedId = item.getAttribute("data-feed-id");
			if (!video) return;

			if (feedId === activeFeedId) {
				video.muted = !hasInteracted;
				video.play().catch(() => {
					video.muted = true;
					hasInteracted = false;
					video.play().catch(() => {});
				});
			} else {
				video.pause();
			}
		});
	});

	function calculateScore(short: any) {
		return "0.0";
	}

	function watchVisibility(node: HTMLElement, { id, feedId }: { id: string; feedId: string }) {
		const observer = new IntersectionObserver(
			(entries) => {
				entries.forEach((entry) => {
					if (entry.isIntersecting) {
						activeFeedId = feedId;
						activeDropdownId = null;

						if (window.location.pathname !== `/shorts/${id}`) {
							window.history.replaceState({}, "", `/shorts/${id}`);
						}

						const currentIndex = feed.findIndex((s) => s.feedId === feedId);
						if (currentIndex >= feed.length - 3) {
							loadShortsBatch();
						}
					}
				});
			},
			{
				root: containerElement,
				threshold: 0.5
			}
		);

		observer.observe(node);

		return {
			destroy() {
				observer.disconnect();
			}
		};
	}

	function toggleLike(feedShort: any) {
		feedShort.liked = !feedShort.liked;

		if (feedShort.liked) {
			const newSparks = [];
			for (let i = 0; i < 24; i++) {
				const angle = Math.random() * Math.PI * 2;
				const distance = 25 + Math.random() * 65;
				newSparks.push({
					id: Math.random(),
					x: Math.cos(angle) * distance,
					y: Math.sin(angle) * distance
				});
			}
			feedShort.sparks = newSparks;

			setTimeout(() => {
				feedShort.sparks = [];
			}, 600);
		}
	}

	async function handleShare(short: any) {
		const shareUrl = `${window.location.origin}/shorts/${short.id}`;
		try {
			await navigator.clipboard.writeText(shareUrl);
			short.copied = true;
			toast("URL copied to clipboard!", undefined, undefined, 2000, "success");
			setTimeout(() => {
				short.copied = false;
			}, 2000);
		} catch (err) {
			console.error("Failed to copy link:", err);
		}
	}

	function toggleFullscreen() {
		if (!containerElement) return;

		if (!document.fullscreenElement) {
			containerElement.requestFullscreen?.().catch((err) => console.error(err));
		} else {
			document.exitFullscreen?.().catch((err) => console.error(err));
		}
		activeDropdownId = null;
	}

	function togglePanel(feedId: string | null, type: string | null) {
		if (activePanel.feedId === feedId && activePanel.type === type) {
			activePanel = { feedId: null, type: null };
		} else {
			activePanel = { feedId, type };
		}
		activeDropdownId = null;
	}
</script>

<div class="shorts-page">
	{#if !appState.isMobile}
		<div class="upload-bar">
			<Flex height="fit-content" justifyContent="end">
				<LinkButton href="/shorts/manage">Manage your videos</LinkButton>
			</Flex>
		</div>
	{/if}

	<div class="shorts-container" bind:this={containerElement}>
		{#if isLoading}
			<Flex justifyContent="center" alignItems="center" height="100%">
				<p>Loading shorts...</p>
			</Flex>
		{/if}

		{#each feed as short, index (short.feedId)}
			{@const isActive = short.feedId === activeFeedId}
			{@const isNear = activeIndex !== -1 && Math.abs(index - activeIndex) <= 2}

			<div
				class="short-item"
				data-feed-id={short.feedId}
				style:opacity={isActive ? "1" : "0.4"}
				use:watchVisibility={{ id: short.id, feedId: short.feedId }}>
				<video
					src={isNear ? short.videoUrl : ""}
					loop
					muted={!hasInteracted}
					playsinline
					preload={Math.abs(index - activeIndex) <= 1 ? "auto" : "metadata"}
					onloadeddata={(e) => {
						const target = e.target as HTMLVideoElement;
						if (target.currentTime === 0) target.currentTime = 0.1;
						if (isActive) {
							target.play().catch(() => {});
						}
					}}
					onclick={(e) => {
						const target = e.target as HTMLVideoElement;
						if (!hasInteracted) {
							hasInteracted = true;
							target.muted = false;
							target.play().catch(() => {});
						} else {
							target.paused ? target.play() : target.pause();
						}
					}}>
				</video>

				<div class="top-menu-wrapper">
					<Dropdown isOpen={activeDropdownId === short.feedId} placement="bottom-end">
						{#snippet trigger()}
							<button
								class="top-menu-btn"
								aria-label="Menu"
								onclick={() => {
									activeDropdownId = activeDropdownId === short.feedId ? null : short.feedId;
								}}>
								<Icon icon="more_vert" />
							</button>
						{/snippet}

						<Button
							iconbefore="legend_toggle"
							appearance="subtle"
							alignContent="left"
							stretchwidth
							onclick={() => togglePanel(short.feedId, "info")}>
							Information
						</Button>
						<Button
							iconbefore="flag"
							appearance="subtle"
							alignContent="left"
							stretchwidth
							onclick={() => {
								reportedShortId = short.id;
								showreporter = true;
								activeDropdownId = null;
							}}>
							Report
						</Button>
						<Button
							iconbefore={isFullscreen ? "fullscreen_exit" : "fullscreen"}
							appearance="subtle"
							alignContent="left"
							stretchwidth
							onclick={toggleFullscreen}>
							{isFullscreen ? "Exit Fullscreen" : "Fullscreen"}
						</Button>
					</Dropdown>
					{#if appState.isMobile}
						<button
							class="top-menu-btn"
							aria-label="Menu"
							onclick={() => {
								goto("/shorts/manage");
							}}>
							<Icon icon="video_template" />
						</button>
					{/if}
				</div>

				<div class="overlay">
					<h3>@{short.creatorDisplayName || short.creator}</h3>
					<p>{short.title}</p>
				</div>

				<div class="action-buttons">
					<div class="btn-wrapper">
						<button class="action-btn" aria-label="Like" onclick={() => toggleLike(short)}>
							{#if short.liked}
								<Icon icon="favorite" color="danger" />
							{:else}
								<Icon icon="favorite" />
							{/if}
						</button>

						{#each short.sparks as spark (spark.id)}
							<span
								class="spark"
								style:--tx="{spark.x}px"
								style:--ty="{spark.y}px"
								style="background-color: {token.theme.color.text.danger}">
							</span>
						{/each}
					</div>

					<button
						class="action-btn"
						aria-label="Comment"
						onclick={() => togglePanel(short.feedId, "comment")}>
						<Icon icon="tooltip_2" />
					</button>
					<button class="action-btn" aria-label="Share" onclick={() => handleShare(short)}>
						{#if short.copied}
							<Icon icon="check" />
						{:else}
							<Icon icon="share" />
						{/if}
					</button>
				</div>

				{#if activePanel.feedId === short.feedId && activePanel.type}
					<div class="side-panel">
						<div class="panel-header">
							<h4>{activePanel.type === "info" ? "Information" : "Comments"}</h4>
							<button
								class="close-panel-btn"
								onclick={() => togglePanel(short.feedId, null)}
								aria-label="Close">
								<Icon icon="close" />
							</button>
						</div>
						<div class="panel-content">
							{#if activePanel.type === "info"}
								<div class="info-content">
									<p>
										<strong>Title:</strong>
										{short.title}
									</p>
									<p>
										<strong>Creator:</strong>
										@{short.creator}
									</p>
									<p>
										<strong>Video ID:</strong>
										<span style="font-size: 0.8em; opacity: 0.7;">{short.id}</span>
									</p>
									<hr class="panel-divider" />
									<p class="score-highlight">
										<strong>Algorithm Score:</strong>
										{calculateScore(short)} pts
									</p>
									<p>
										<strong>Views:</strong>
										{short.views.toLocaleString()}
									</p>
									<p>
										<strong>Likes:</strong>
										{short.likesCount + (short.liked ? 1 : 0)}
									</p>
									<p>
										<strong>Watch Duration:</strong>
										{short.watchDuration}s
									</p>
									<p>
										<strong>Video Length:</strong>
										{short.videoLength}s
									</p>
								</div>
							{:else if activePanel.type === "comment"}
								<div class="empty-comments">
									<p style="text-align: center; opacity: 0.6; margin-top: 2rem;">
										<img src="https://home.davidnet.net/operation-teapot-nuke.gif" alt="" />
									</p>
								</div>
							{/if}
						</div>
					</div>
				{/if}
			</div>
		{/each}
	</div>
</div>

{#if showreporter && reportedShortId}
	<ReportModal bind:isOpen={showreporter} reportType="short" reportedId={reportedShortId} />
{/if}

<style>
	.shorts-page {
		position: relative;
		width: 100%;
		height: calc(100dvh - 56px);
	}

	.upload-bar {
		position: absolute;
		top: 16px;
		right: 16px;
		z-index: 50;
	}

	.shorts-container {
		height: 100%;
		width: 100%;
		overflow-y: scroll;
		scroll-snap-type: y mandatory;
		display: flex;
		flex-direction: column;
		align-items: center;
		-webkit-overflow-scrolling: touch;
	}

	.shorts-container:fullscreen {
		background-color: #000;
		height: 100dvh;
		width: 100vw;
		display: flex;
		flex-direction: column;
		align-items: center;
		overflow-y: scroll;
		scroll-snap-type: y mandatory;
	}

	.shorts-container::backdrop {
		background-color: #000;
	}

	.short-item {
		flex-shrink: 0;
		height: 100%;
		aspect-ratio: 9 / 16;
		scroll-snap-align: start;
		scroll-snap-stop: always;
		position: relative;
		display: flex;
		justify-content: center;
		align-items: center;
		border-radius: 12px;
		overflow: hidden;
		transition: opacity 0.3s ease;
		background-color: #121212;
	}

	video {
		width: 100%;
		height: 100%;
		object-fit: cover;
	}

	.top-menu-wrapper {
		position: absolute;
		top: 16px;
		right: 16px;
		z-index: 20;
		display: flex;
		width: fit-content;
		gap: 0.5rem;
		flex-direction: column;
	}

	.top-menu-btn {
		background: rgba(0, 0, 0, 0.5);
		border: none;
		color: white;
		width: 44px;
		height: 44px;
		border-radius: 50%;
		cursor: pointer;
		display: flex;
		justify-content: center;
		align-items: center;
		font-size: 1.5rem;
	}

	.overlay {
		position: absolute;
		bottom: 24px;
		left: 16px;
		right: 80px;
		color: white;
		text-shadow: 0 2px 4px rgba(0, 0, 0, 0.8);
		pointer-events: none;
		z-index: 10;
	}

	.overlay h3 {
		margin: 0 0 4px 0;
		font-size: 1.1rem;
	}

	.overlay p {
		margin: 0;
		font-size: 0.95rem;
		opacity: 0.9;
	}

	.action-buttons {
		position: absolute;
		right: 16px;
		bottom: 24px;
		display: flex;
		flex-direction: column;
		gap: 16px;
		align-items: center;
		z-index: 10;
	}

	.btn-wrapper {
		position: relative;
		display: flex;
		justify-content: center;
		align-items: center;
	}

	.action-btn {
		background: rgba(0, 0, 0, 0.6);
		border: none;
		color: white;
		width: 52px;
		height: 52px;
		border-radius: 50%;
		cursor: pointer;
		display: flex;
		justify-content: center;
		align-items: center;
		font-size: 1.3rem;
		transition: transform 0.2s;
	}

	.action-btn:hover {
		transform: scale(1.1);
	}

	.spark {
		position: absolute;
		width: 6px;
		height: 6px;
		border-radius: 50%;
		pointer-events: none;
		animation: spark-fly 0.6s ease-out forwards;
	}

	.side-panel {
		position: absolute;
		top: 0;
		right: 0;
		width: 100%;
		height: 100%;
		background: rgba(18, 18, 18, 0.95);
		backdrop-filter: blur(12px);
		-webkit-backdrop-filter: blur(12px);
		z-index: 30;
		display: flex;
		flex-direction: column;
		padding: 24px;
		box-sizing: border-box;
		color: white;
		animation: slideIn 0.25s ease-out;
	}

	@keyframes slideIn {
		from {
			transform: translateX(100%);
		}
		to {
			transform: translateX(0);
		}
	}

	.panel-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		border-bottom: 1px solid rgba(255, 255, 255, 0.15);
		padding-bottom: 12px;
		margin-bottom: 16px;
	}

	.panel-header h4 {
		margin: 0;
		font-size: 1.1rem;
	}

	.close-panel-btn {
		background: transparent;
		border: none;
		color: white;
		cursor: pointer;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 4px;
		border-radius: 50%;
	}

	.close-panel-btn:hover {
		background: rgba(255, 255, 255, 0.1);
	}

	.panel-content {
		flex: 1;
		overflow-y: auto;
	}

	.info-content p {
		margin: 8px 0;
		font-size: 0.95rem;
		color: rgba(255, 255, 255, 0.85);
	}

	.panel-divider {
		border: 0;
		border-top: 1px solid rgba(255, 255, 255, 0.15);
		margin: 16px 0;
	}

	.score-highlight {
		font-size: 1.1rem !important;
		color: #00f2fe !important;
	}

	@keyframes spark-fly {
		0% {
			transform: translate(0, 0) scale(1);
			opacity: 1;
		}
		100% {
			transform: translate(var(--tx), var(--ty)) scale(0);
			opacity: 0;
		}
	}
</style>
