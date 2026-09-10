<script>
	import { page } from "$app/state";
	import { onMount } from "svelte";
	import { Icon } from "@davidnet-net/svelte-ui";
	import { token } from "@davidnet-net/svelte-ui/tokens";

	let shorts = $state([
		{
			id: 1,
			title: "Eerste Short",
			videoUrl: "/test_videos/video1.mp4",
			creator: "Davidnet",
			liked: false,
			sparks: []
		},
		{
			id: 2,
			title: "Tweede Short",
			videoUrl: "/test_videos/video2.mp4",
			creator: "Davidnet",
			liked: false,
			sparks: []
		},
		{
			id: 3,
			title: "Derde Short",
			videoUrl: "/test_videos/video3.mp4",
			creator: "Davidnet",
			liked: false,
			sparks: []
		},
		{
			id: 4,
			title: "Vierde Short",
			videoUrl: "/test_videos/video4.mp4",
			creator: "Davidnet",
			liked: false,
			sparks: []
		},
		{
			id: 5,
			title: "Vijfde Short",
			videoUrl: "/test_videos/video5.mp4",
			creator: "Davidnet",
			liked: false,
			sparks: []
		},
		{
			id: 6,
			title: "Zesde Short",
			videoUrl: "/test_videos/video6.mp4",
			creator: "Davidnet",
			liked: false,
			sparks: []
		},
		{
			id: 7,
			title: "Zevende Short",
			videoUrl: "/test_videos/video7.mp4",
			creator: "Davidnet",
			liked: false,
			sparks: []
		},
		{
			id: 8,
			title: "8de Short",
			videoUrl: "/test_videos/video8.mp4",
			creator: "Davidnet",
			liked: false,
			sparks: []
		},
		{
			id: 9,
			title: "Bimbambini sixseveni",
			videoUrl: "/test_videos/video9.mp4",
			creator: "Davidnet",
			liked: false,
			sparks: []
		}
	]);

	let activeId = $state(1);
	let videoElements = {};
	let containerElement;

	// Reactief effect: speel alleen de actieve video af en pauzeer de rest
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
				// Zet de scrollpositie van de container direct op het juiste element
				containerElement.scrollTop = targetItem.offsetTop;
			}
		}
	});

	function watchVisibility(node, id) {
		const observer = new IntersectionObserver(
			(entries) => {
				entries.forEach((entry) => {
					if (entry.isIntersecting) {
						activeId = id;
						// Pas de URL passief aan als deze afwijkt
						if (window.location.pathname !== `/shorts/${id}`) {
							window.history.replaceState({}, "", `/shorts/${id}`);
						}
					}
				});
			},
			{
				root: containerElement, // Koppel observer direct aan de scrollcontainer
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
</script>

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

			<!-- 3 Dots menu button at the top -->
			<button class="top-menu-btn" aria-label="Menu"><Icon icon="more_vert" /></button>

			<!-- Overlay text -->
			<div class="overlay">
				<h3>@{short.creator}</h3>
				<p>{short.title} (ID: {short.id})</p>
			</div>

			<!-- Thumb zone action buttons (Right side) -->
			<div class="action-buttons">
				<div class="btn-wrapper">
					<button class="action-btn" aria-label="Like" onclick={() => toggleLike(short)}>
						{#if short.liked}
							<Icon icon="favorite" color="danger" />
						{:else}
							<Icon icon="favorite" />
						{/if}
					</button>

					<!-- Red sparks animation elements -->
					{#each short.sparks as spark (spark.id)}
						<span
							class="spark"
							style:--tx="{spark.x}px"
							style:--ty="{spark.y}px"
							style="background-color: {token.theme.color.text.danger}">
						</span>
					{/each}
				</div>

				<button class="action-btn" aria-label="Comment"><Icon icon="tooltip_2" /></button>
				<button class="action-btn" aria-label="Share"><Icon icon="share" /></button>
			</div>
		</div>
	{/each}
</div>

<style>
	.shorts-container {
		height: calc(100vh - 56px);
		width: 100%;
		overflow-y: scroll;
		scroll-snap-type: y mandatory;
		display: flex;
		flex-direction: column;
		align-items: center;
	}

	.short-item {
		width: 100%;
		max-width: 600px;
		height: 100%;
		min-height: 100%;
		scroll-snap-align: start;
		scroll-snap-stop: always;
		position: relative;
		display: flex;
		justify-content: center;
		align-items: center;
		transition: opacity 0.3s ease;
	}

	video {
		width: 100%;
		height: 100%;
		object-fit: cover;
	}

	.top-menu-btn {
		position: absolute;
		top: 20px;
		right: 20px;
		background: rgba(0, 0, 0, 0.5);
		border: none;
		color: white;
		width: 40px;
		height: 40px;
		border-radius: 50%;
		cursor: pointer;
		display: flex;
		justify-content: center;
		align-items: center;
		font-size: 1.5rem;
		z-index: 10;
	}

	.overlay {
		position: absolute;
		bottom: 40px;
		left: 20px;
		right: 80px;
		color: white;
		text-shadow: 0 2px 4px rgba(0, 0, 0, 0.8);
		pointer-events: none;
	}

	.action-buttons {
		position: absolute;
		right: 16px;
		bottom: 40px;
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
		width: 48px;
		height: 48px;
		border-radius: 50%;
		cursor: pointer;
		display: flex;
		justify-content: center;
		align-items: center;
		font-size: 1.2rem;
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
