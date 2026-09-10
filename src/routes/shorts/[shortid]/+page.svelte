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

	let shorts = $state([
		{
			id: 1,
			title: "Eerste Short",
			videoUrl: "/test_videos/video1.mp4",
			creator: "Davidnet",
			liked: false,
			sparks: [],
			copied: false,
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
			sparks: [],
			copied: false,
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
			sparks: [],
			copied: false,
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
			sparks: [],
			copied: false,
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
			sparks: [],
			copied: false,
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
			sparks: [],
			copied: false,
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
			sparks: [],
			copied: false,
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
			sparks: [],
			copied: false,
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
			sparks: [],
			copied: false,
			views: 6700,
			likesCount: 890,
			watchDuration: 58,
			videoLength: 60
		}
	]);

	let activeId = $state(1);
	let activeDropdownId = $state(null);
	let activePanel = $state({ id: null, type: null });
	let isFullscreen = $state(false);
	let videoElements = {};
	let containerElement;

	function calculateScore(short) {
		if (!short.videoLength) return "0.0";
		const totalLikes = short.likesCount + (short.liked ? 1 : 0);
		const score = (short.watchDuration / short.videoLength) * 100 + totalLikes * 2;
		return score.toFixed(1);
	}

	$effect(() => {
		Object.entries(videoElements).forEach(([id, video]) => {
			if (!video) return;
			if (Number(id) === activeId) {
				video.play().catch(() => {});
			} else {
				video.pause();
			}
		});
	});

	onMount(() => {
		const initialId = Number(page.params.shortid);
		if (initialId && containerElement) {
			const targetItem = containerElement.querySelector(`[data-id="${initialId}"]`);
			if (targetItem) {
				activeId = initialId;
				containerElement.scrollTop = targetItem.offsetTop;
			}
		}

		const handleFsChange = () => {
			isFullscreen = !!document.fullscreenElement;
		};
		document.addEventListener("fullscreenchange", handleFsChange);
		return () => document.removeEventListener("fullscreenchange", handleFsChange);
	});

	function watchVisibility(node, id) {
		const observer = new IntersectionObserver(
			(entries) => {
				entries.forEach((entry) => {
					if (entry.isIntersecting) {
						activeId = id;
						activeDropdownId = null;
						if (window.location.pathname !== `/shorts/${id}`) {
							window.history.replaceState({}, "", `/shorts/${id}`);
						}
					}
				});
			},
			{
				root: containerElement,
				threshold: 0.6
			}
		);

		observer.observe(node);

		return {
			destroy() {
				observer.disconnect();
			}
		};
	}

	function toggleLike(short) {
		short.liked = !short.liked;
		if (short.liked) {
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
			short.sparks = newSparks;

			setTimeout(() => {
				short.sparks = [];
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

	function toggleFullscreen(shortId) {
		const targetItem = containerElement?.querySelector(`[data-id="${shortId}"]`);
		if (!targetItem) return;

		if (!document.fullscreenElement) {
			targetItem.requestFullscreen?.().catch((err) => console.error(err));
		} else {
			document.exitFullscreen?.().catch((err) => console.error(err));
		}
		activeDropdownId = null;
	}

	function togglePanel(shortId, type) {
		if (activePanel.id === shortId && activePanel.type === type) {
			activePanel = { id: null, type: null };
		} else {
			activePanel = { id: shortId, type };
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
		{#each shorts as short (short.id)}
			<div
				class="short-item"
				data-id={short.id}
				style:opacity={activeId === short.id ? "1" : "0.4"}
				use:watchVisibility={short.id}>
				<video
					bind:this={videoElements[short.id]}
					src={short.videoUrl}
					loop
					muted
					playsinline
					preload="metadata"
					onclick={(e) => (e.target.paused ? e.target.play() : e.target.pause())}>
				</video>

				<!-- Menu Dropdown -->
				<div class="top-menu-wrapper">
					<Dropdown isOpen={activeDropdownId === short.id} placement="bottom-end">
						{#snippet trigger()}
							<button
								class="top-menu-btn"
								aria-label="Menu"
								onclick={() => {
									activeDropdownId = activeDropdownId === short.id ? null : short.id;
								}}>
								<Icon icon="more_vert" />
							</button>
						{/snippet}

						<Button
							iconbefore="legend_toggle"
							appearance="subtle"
							alignContent="left"
							stretchwidth
							onclick={() => togglePanel(short.id, "info")}>
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
							onclick={() => toggleFullscreen(short.id)}>
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

				<!-- Overlay text -->
				<div class="overlay">
					<h3>@{short.creator}</h3>
					<p>{short.title} (ID: {short.id})</p>
				</div>

				<!-- Action buttons -->
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
						onclick={() => togglePanel(short.id, "comment")}>
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

				<!-- Side Panel (Information & Comments) -->
				{#if activePanel.id === short.id && activePanel.type}
					<div class="side-panel">
						<div class="panel-header">
							<h4>{activePanel.type === "info" ? "Information" : "Comments"}</h4>
							<button
								class="close-panel-btn"
								onclick={() => togglePanel(short.id, null)}
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
	}

	/* Fullscreen ondersteuning */
	.shorts-container:fullscreen {
		background-color: #000;
		height: 100dvh;
		width: 100vw;
	}

	.shorts-container::backdrop {
		background-color: #000;
	}

	.short-item {
		/* CRUCIAAL: Voorkomt dat flexbox alle items samenknijpt in 1 scherm */
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
