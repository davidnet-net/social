<script>
	import { page } from "$app/state";
	import { onMount } from "svelte";
	import {
		appState,
		Button,
		Dropdown,
		Flex,
		Icon,
		LinkButton,
		toast
	} from "@davidnet-net/svelte-ui";
	import { token } from "@davidnet-net/svelte-ui/tokens";
	import { goto } from "$app/navigation";

	let sourceShorts = $state([
		{
			id: 1,
			title: "Eerste Short",
			videoUrl: "/test_videos/video1.mp4",
			creator: "Davidnet",
			liked: false,
			views: 1420,
			likesCount: 112,
			watchDuration: 18,
			videoLength: 20
		},
		{
			id: 2,
			title: "Tweede Short",
			videoUrl: "/test_videos/video2.mp4",
			creator: "Davidnet",
			liked: false,
			views: 890,
			likesCount: 64,
			watchDuration: 12,
			videoLength: 15
		},
		{
			id: 3,
			title: "Derde Short",
			videoUrl: "/test_videos/video3.mp4",
			creator: "Davidnet",
			liked: false,
			views: 2310,
			likesCount: 310,
			watchDuration: 28,
			videoLength: 30
		},
		{
			id: 4,
			title: "Vierde Short",
			videoUrl: "/test_videos/video4.mp4",
			creator: "Davidnet",
			liked: false,
			views: 540,
			likesCount: 22,
			watchDuration: 8,
			videoLength: 10
		},
		{
			id: 5,
			title: "Vijfde Short",
			videoUrl: "/test_videos/video5.mp4",
			creator: "Davidnet",
			liked: false,
			views: 1100,
			likesCount: 95,
			watchDuration: 22,
			videoLength: 25
		},
		{
			id: 6,
			title: "Zesde Short",
			videoUrl: "/test_videos/video6.mp4",
			creator: "Davidnet",
			liked: false,
			views: 340,
			likesCount: 18,
			watchDuration: 5,
			videoLength: 12
		},
		{
			id: 7,
			title: "Zevende Short",
			videoUrl: "/test_videos/video7.mp4",
			creator: "Davidnet",
			liked: false,
			views: 4200,
			likesCount: 512,
			watchDuration: 45,
			videoLength: 45
		},
		{
			id: 8,
			title: "8de Short",
			videoUrl: "/test_videos/video8.mp4",
			creator: "Davidnet",
			liked: false,
			views: 780,
			likesCount: 45,
			watchDuration: 10,
			videoLength: 14
		},
		{
			id: 9,
			title: "Bimbambini sixseveni",
			videoUrl: "/test_videos/video9.mp4",
			creator: "Davidnet",
			liked: false,
			views: 6700,
			likesCount: 890,
			watchDuration: 58,
			videoLength: 60
		}
	]);

	let feed = $state([]);
	let activeFeedId = $state(null);
	let activeDropdownId = $state(null);
	let activePanel = $state({ feedId: null, type: null });
	let isFullscreen = $state(false);
	let containerElement;

	function createBatch(firstId = null) {
		let shuffled = [...sourceShorts].sort(() => Math.random() - 0.5);

		if (firstId) {
			const targetIndex = shuffled.findIndex((s) => s.id === firstId);
			if (targetIndex > -1) {
				const [target] = shuffled.splice(targetIndex, 1);
				shuffled.unshift(target);
			}
		}

		return shuffled.map((s) => ({
			...s,
			feedId: Math.random().toString(36).substring(2, 11),
			sparks: [],
			copied: false
		}));
	}

	onMount(() => {
		const initialId = Number(page.params.shortid);
		// Laad direct meerdere batches achter elkaar zodat de buffer meteen gevuld is met ~18 items
		feed = [...createBatch(initialId), ...createBatch()];

		if (feed.length > 0) {
			activeFeedId = feed[0].feedId;
		}

		if (initialId && containerElement) {
			const targetItem = containerElement.querySelector(`[data-feed-id="${activeFeedId}"]`);
			if (targetItem) {
				containerElement.scrollTop = targetItem.offsetTop;
			}
		}

		const handleFsChange = () => {
			isFullscreen = !!document.fullscreenElement;
		};
		document.addEventListener("fullscreenchange", handleFsChange);
		return () => document.removeEventListener("fullscreenchange", handleFsChange);
	});

	function calculateScore(short) {
		if (!short.videoLength) return "0.0";
		const totalLikes = short.likesCount + (short.liked ? 1 : 0);
		const score = (short.watchDuration / short.videoLength) * 100 + totalLikes * 2;
		return score.toFixed(1);
	}

	function watchVisibility(node, { id, feedId }) {
		const observer = new IntersectionObserver(
			(entries) => {
				entries.forEach((entry) => {
					const video = node.querySelector("video");

					if (entry.isIntersecting) {
						activeFeedId = feedId;
						activeDropdownId = null;

						if (window.location.pathname !== `/shorts/${id}`) {
							window.history.replaceState({}, "", `/shorts/${id}`);
						}

						video?.play().catch(() => {});

						const currentIndex = feed.findIndex((s) => s.feedId === feedId);
						// Al bij 10 items van tevoren nieuwe batches inladen en klaarzetten
						if (currentIndex >= feed.length - 10) {
							feed = [...feed, ...createBatch(), ...createBatch()];
						}
					} else {
						video?.pause();
					}
				});
			},
			{
				root: containerElement,
				threshold: 0.4
			}
		);

		observer.observe(node);

		return {
			destroy() {
				observer.disconnect();
			}
		};
	}

	function toggleLike(feedShort) {
		const sourceShort = sourceShorts.find((s) => s.id === feedShort.id);
		if (sourceShort) sourceShort.liked = !sourceShort.liked;

		feed.forEach((item) => {
			if (item.id === feedShort.id) item.liked = sourceShort.liked;
		});

		if (sourceShort && sourceShort.liked) {
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

	async function handleShare(short) {
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

	function togglePanel(feedId, type) {
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
		{#each feed as short (short.feedId)}
			<div
				class="short-item"
				data-feed-id={short.feedId}
				style:opacity={activeFeedId === short.feedId ? "1" : "0.4"}
				use:watchVisibility={{ id: short.id, feedId: short.feedId }}>
				<video
					src={short.videoUrl}
					loop
					muted
					playsinline
					preload="auto"
					onloadedmetadata={(e) => {
						e.target.currentTime = 0.1;
					}}
					onclick={(e) => (e.target.paused ? e.target.play() : e.target.pause())}>
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
						<Button iconbefore="flag" appearance="subtle" alignContent="left" stretchwidth>
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
					<h3>@{short.creator}</h3>
					<p>{short.title} (ID: {short.id})</p>
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
										{short.id}
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
								<div class="empty-comments"></div>
							{/if}
						</div>
					</div>
				{/if}
			</div>
		{/each}
	</div>
</div>

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
