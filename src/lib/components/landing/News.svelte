<script>
	import { pool } from '$lib/stores/nostr-infrastructure.svelte';
	import { nip19 } from 'nostr-tools';
	import { marked } from 'marked';

	// Configuration für verschiedene Content-Publisher
	const publishers = {
		efabiResources: {
			name: 'efabi Ressourcen',
			npub: 'npub14esruzfvraksa9gyqkyp0uzgp5ytnlqu96rpj0ergk05s9hy9wrspsljvc',
			kind: 30023
		},
		efabiEvents: {
			name: 'efabi Veranstaltungen',
			npub: 'npub14esruzfvraksa9gyqkyp0uzgp5ytnlqu96rpj0ergk05s9hy9wrspsljvc',
			kind: 31923
		},
		efabiNetwork: {
			name: 'efabi Netzwerkstatt',
			npub: 'npub14esruzfvraksa9gyqkyp0uzgp5ytnlqu96rpj0ergk05s9hy9wrspsljvc',
			kind: 0
		}
	};

	const relays = [
		'wss://relay-rpi.edufeed.org/',
		'wss://relay.primal.net/',
	];

	// Setup marked options
	marked.setOptions({
		breaks: true,
		gfm: true
	});

	// State variables
	let isLoading = $state(true);
	let isConnected = $state(false);
	let connectedRelays = $state(0);

	// Content arrays
	/** @type {any[]} */
	let efabiResources = $state([]);
	/** @type {any[]} */
	let efabiEvents = $state([]);
	/** @type {any[]} */
	let efabiNetwork = $state([]);

	// Carousel variables
	/** @type {HTMLElement | null} */
	let carouselTrack = $state(null);
	let currentScrollPosition = $state(0);

    // Load content on component mount
    loadContent();

	/**
	 * @param {number} timestamp
	 */
	function formatDate(timestamp) {
		const date = new Date(timestamp * 1000);
		return date.toLocaleDateString('de-DE', { 
			year: 'numeric', 
			month: 'long', 
			day: 'numeric' 
		});
	}

	/**
	 * @param {number} timestamp
	 */
	function formatDateTime(timestamp) {
		const date = new Date(timestamp * 1000);
		return date.toLocaleDateString('de-DE', { 
			year: 'numeric', 
			month: 'long', 
			day: 'numeric',
			hour: '2-digit',
			minute: '2-digit'
		});
	}

	/**
	 * Scroll carousel to previous items
	 */
	function scrollPrev() {
		if (!carouselTrack) return;
		const cardWidth = 320; // Card width + gap
		const scrollAmount = cardWidth * 3; // Scroll by 3 cards
		currentScrollPosition = Math.max(0, currentScrollPosition - scrollAmount);
		carouselTrack.scrollTo({
			left: currentScrollPosition,
			behavior: 'smooth'
		});
	}

	/**
	 * Scroll carousel to next items
	 */
	function scrollNext() {
		if (!carouselTrack) return;
		const cardWidth = 320; // Card width + gap
		const scrollAmount = cardWidth * 3; // Scroll by 3 cards
		const maxScroll = carouselTrack.scrollWidth - carouselTrack.clientWidth;
		currentScrollPosition = Math.min(maxScroll, currentScrollPosition + scrollAmount);
		carouselTrack.scrollTo({
			left: currentScrollPosition,
			behavior: 'smooth'
		});
	}

	async function loadContent() {
		try {
			console.log('🔄 Starte Laden der Inhalte...');
			
			isConnected = true;
			connectedRelays = relays.length;
			console.log('🔗 Verbinde zu Relays:', relays);

			// Decode publisher pubkeys
			const { data: efabiHex } = nip19.decode(publishers.efabiResources.npub);

			console.log('📊 EFABI Hex:', efabiHex);

			// Load EFABI Ressourcen (NIP-23: Long-form Content)
			const efabiResourceFilter = {
				kinds: [publishers.efabiResources.kind],
				authors: [efabiHex],
				limit: 50
			};
			console.log('🔍 EFABI Ressourcen-Filter:', efabiResourceFilter);

			// Load EFABI Veranstaltungen (NIP-52: Calendar Events)
			const efabiEventFilter = {
				kinds: [publishers.efabiEvents.kind],
				authors: [efabiHex],
				limit: 50
			};
			console.log('🔍 EFABI Event-Filter:', efabiEventFilter);

			// Load EFABI Netzwerkstatt Profile (NIP-05: User Metadata)
			const efabiNetworkFilter = {
				kinds: [publishers.efabiNetwork.kind],
				authors: [efabiHex],
				limit: 50
			};
			console.log('🔍 EFABI Netzwerkstatt-Filter:', efabiNetworkFilter);

			// Load content using pool subscriptions
			/**
			 * @param {any} filter
			 */
			const loadData = (filter) => {
				return new Promise((resolve) => {
					/** @type {any[]} */
					const events = [];
					const sub = pool
						.group(relays)
						.subscription(filter)
						.subscribe({
							next: (response) => {
								if (response === 'EOSE') {
									resolve(events);
									sub.unsubscribe();
								} else if (response && typeof response === 'object' && response.id) {
									events.push(response);
								}
							},
							error: (error) => {
								console.error('Subscription error:', error);
								resolve(events);
							}
						});
				});
			};

			// Query all filters in parallel
			const [
				efabiResourcesResult,
				efabiEventsResult,
				efabiNetworkResult
			] = await Promise.all([
				loadData(efabiResourceFilter),
				loadData(efabiEventFilter),
				loadData(efabiNetworkFilter)
			]);

			efabiResources = efabiResourcesResult.sort((a, b) => b.created_at - a.created_at) || [];
			efabiEvents = efabiEventsResult.sort((a, b) => {
                        const aStart = a.tags.find((/** @type {any} */ t) => t[0] === 'start')?.[1] || a.created_at;
                        const bStart = b.tags.find((/** @type {any} */ t) => t[0] === 'start')?.[1] || b.created_at;
                        return parseInt(aStart) - parseInt(bStart);
                    }) || [];
			efabiNetwork = efabiNetworkResult || [];

			console.log('✅ EFABI Ressourcen geladen:', efabiResources.length);
			console.log('✅ EFABI Events geladen:', efabiEvents.length);
			console.log('✅ EFABI Netzwerk-Profile geladen:', efabiNetwork.length);

			isLoading = false;
			
		} catch (error) {
			console.error('❌ Fehler beim Laden der Inhalte:', error);
			isLoading = false;
			isConnected = false;
		}
	}
</script>

<!-- EFABI Content Portal: For non-logged-in users -->
<div class="container">
    {#if isLoading}
        <div class="loading">Lade Inhalte...</div>
    {:else}
        <!-- Content Carousel -->
        <div class="carousel-container">
            <div class="carousel-header">
                <h2 class="carousel-title" style="color: oklch(80% 38% 76deg)">Neuigkeiten</h2>
                <div class="carousel-controls">
                    <button class="carousel-btn prev" onclick={scrollPrev} aria-label="Previous">‹</button>
                    <button class="carousel-btn next" onclick={scrollNext} aria-label="Next">›</button>
                </div>
            </div>
            
            <div class="carousel-wrapper">
                <div class="carousel-track" bind:this={carouselTrack}>
                    <!-- EFABI Resources Cards -->
                    {#each efabiResources as article}
                        {@const title = article.tags.find((/** @type {any} */ t) => t[0] === 'title')?.[1] || 'Ohne Titel'}
                        {@const summary = article.tags.find((/** @type {any} */ t) => t[0] === 'summary')?.[1] || ''}
                        {@const image = article.tags.find((/** @type {any} */ t) => t[0] === 'image')?.[1] || ''}
                        {@const publishedAt = article.tags.find((/** @type {any} */ t) => t[0] === 'published_at')?.[1]}
                        {@const displayDate = publishedAt ? parseInt(publishedAt) : article.created_at}
                        
                        <div class="card article-card">
                            <div class="card-badge">📝 efabi Artikel</div>
                            {#if image}
                                <img src={image} alt={title} class="card-image" />
                            {/if}
                            <div class="card-content">
                                <h3 class="card-title">{title}</h3>
                                {#if summary}
                                    <p class="card-summary">{summary}</p>
                                {/if}
                                <div class="card-meta">
                                    <span class="meta-icon">📅</span>
                                    <span>{formatDate(displayDate)}</span>
                                </div>
                            </div>
                        </div>
                    {/each}
                    
                    <!-- EFABI Events Cards -->
                    {#each efabiEvents as event}
                        {@const title = event.tags.find((/** @type {any} */ t) => t[0] === 'title')?.[1] || 'Ohne Titel'}
                        {@const summary = event.tags.find((/** @type {any} */ t) => t[0] === 'summary')?.[1] || ''}
                        {@const location = event.tags.find((/** @type {any} */ t) => t[0] === 'location')?.[1] || ''}
                        {@const startTime = event.tags.find((/** @type {any} */ t) => t[0] === 'start')?.[1]}
                        {@const image = event.tags.find((/** @type {any} */ t) => t[0] === 'image')?.[1] || ''}
                        
                        <div class="card event-card">
                            <div class="card-badge">📆 efabi Veranstaltung</div>
                            {#if image}
                                <img src={image} alt={title} class="card-image" />
                            {/if}
                            <div class="card-content">
                                <h3 class="card-title">{title}</h3>
                                {#if summary}
                                    <p class="card-summary">{summary}</p>
                                {/if}
                                {#if startTime}
                                    <div class="card-meta">
                                        <span class="meta-icon">🕐</span>
                                        <span>{formatDateTime(parseInt(startTime))}</span>
                                    </div>
                                {/if}
                                {#if location}
                                    <div class="card-meta">
                                        <span class="meta-icon">📍</span>
                                        <span>{location}</span>
                                    </div>
                                {/if}
                            </div>
                        </div>
                    {/each}
                    
                    <!-- EFABI Network Cards -->
                    {#each efabiNetwork as profile}
                        {@const profileData = JSON.parse(profile.content || '{}')}
                        
                        <div class="card profile-card">
                            <div class="card-badge">👤 efabi Netzwerk</div>
                            <div class="card-content">
                                <div class="profile-header-small">
                                    {#if profileData.picture}
                                        <img src={profileData.picture} alt={profileData.name || 'Profil'} class="profile-avatar-small" />
                                    {:else}
                                        <div class="profile-avatar-placeholder-small">👤</div>
                                    {/if}
                                    <div class="profile-info-small">
                                        {#if profileData.name}
                                            <h3 class="card-title profile-name-small">{profileData.name}</h3>
                                        {/if}
                                        {#if profileData.nip05}
                                            <div class="profile-nip05-small">{profileData.nip05}</div>
                                        {/if}
                                    </div>
                                </div>
                                {#if profileData.about}
                                    <p class="card-summary">{profileData.about.slice(0, 120)}...</p>
                                {/if}
                                {#if profileData.website}
                                    <div class="card-meta">
                                        <span class="meta-icon">🌐</span>
                                        <span class="profile-website-small">{profileData.website.replace(/^https?:\/\//, '')}</span>
                                    </div>
                                {/if}
                            </div>
                        </div>
                    {/each}
                </div>
            </div>
        </div>
    {/if}
</div>

<style>
	/* EFABI Design System Colors */
	:global(:root) {
		--efabi-text: #3A4F66;
		--efabi-h1: #FAAC00;
		--efabi-h2: #192A3D;
		--efabi-h3: #3A4F66;
		--efabi-link: #3F40F7;
		--efabi-link-hover: #005D8F;
		--efabi-border: #E1E8ED;
		--efabi-selection-bg: #1559ED;
	}

	.container {
		max-width: 1200px;
		margin: 0 auto;
		padding: 20px;
	}

	/* Carousel Styles */
	.carousel-container {
		background: white;
		border-radius: 16px;
		box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
		padding: 30px;
		margin-bottom: 30px;
	}

	.carousel-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 25px;
		padding-bottom: 15px;
		border-bottom: 2px solid var(--efabi-border);
	}

	.carousel-title {
		color: var(--efabi-h2);
		font-size: 2em;
		font-weight: 700;
		margin: 0;
	}

	.carousel-controls {
		display: flex;
		gap: 10px;
	}

	.carousel-btn {
		width: 50px;
		height: 50px;
		border-radius: 50%;
		border: 2px solid var(--efabi-link);
		background: white;
		color: var(--efabi-link);
		cursor: pointer;
		display: flex;
		align-items: center;
		justify-content: center;
		font-size: 24px;
		font-weight: bold;
		transition: all 0.3s ease;
		box-shadow: 0 2px 8px rgba(63, 64, 247, 0.1);
	}

	.carousel-btn:hover {
		background: var(--efabi-link);
		color: white;
		transform: scale(1.1);
		box-shadow: 0 4px 12px rgba(63, 64, 247, 0.3);
	}

	.carousel-wrapper {
		overflow: hidden;
		border-radius: 12px;
	}

	.carousel-track {
		display: flex;
		gap: 20px;
		overflow-x: auto;
		scroll-behavior: smooth;
		padding-bottom: 10px;
		scrollbar-width: thin;
		scrollbar-color: var(--efabi-border) transparent;
	}

	.carousel-track::-webkit-scrollbar {
		height: 8px;
	}

	.carousel-track::-webkit-scrollbar-track {
		background: var(--efabi-border);
		border-radius: 4px;
	}

	.carousel-track::-webkit-scrollbar-thumb {
		background: var(--efabi-link);
		border-radius: 4px;
	}

	.carousel-track::-webkit-scrollbar-thumb:hover {
		background: var(--efabi-link-hover);
	}

	/* Card Styles */
	.card {
		flex: 0 0 300px;
		background: white;
		border-radius: 12px;
		box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
		border: 1px solid var(--efabi-border);
		transition: all 0.3s ease;
		overflow: hidden;
		position: relative;
	}

	.card:hover {
		transform: translateY(-5px);
		box-shadow: 0 8px 25px rgba(0, 0, 0, 0.15);
		border-color: var(--efabi-link);
	}

	.card-badge {
		position: absolute;
		top: 12px;
		left: 12px;
		background: var(--efabi-link);
		color: white;
		padding: 6px 12px;
		border-radius: 20px;
		font-size: 0.8em;
		font-weight: 600;
		z-index: 2;
		box-shadow: 0 2px 8px rgba(63, 64, 247, 0.3);
	}

	.card-image {
		width: 100%;
		height: 180px;
		object-fit: cover;
		border-bottom: 1px solid var(--efabi-border);
	}

	.card-content {
		padding: 20px;
	}

	.card-title {
		color: var(--efabi-h2);
		font-size: 1.3em;
		font-weight: 700;
		margin-bottom: 12px;
		line-height: 1.3;
		display: -webkit-box;
		-webkit-line-clamp: 2;
		-webkit-box-orient: vertical;
		overflow: hidden;
	}

	.card-summary {
		color: var(--efabi-text);
		font-size: 0.95em;
		line-height: 1.5;
		margin-bottom: 15px;
		display: -webkit-box;
		-webkit-line-clamp: 3;
		-webkit-box-orient: vertical;
		overflow: hidden;
	}

	.card-meta {
		display: flex;
		align-items: center;
		gap: 8px;
		color: var(--efabi-text);
		font-size: 0.85em;
		margin-bottom: 8px;
		opacity: 0.8;
	}

	.card-meta:last-child {
		margin-bottom: 0;
	}

	.meta-icon {
		font-size: 1em;
		flex-shrink: 0;
	}

	/* Profile Card Specific Styles */
	.profile-header-small {
		display: flex;
		gap: 15px;
		align-items: center;
		margin-bottom: 15px;
	}

	.profile-avatar-small {
		width: 60px;
		height: 60px;
		border-radius: 50%;
		object-fit: cover;
		border: 3px solid var(--efabi-link);
		flex-shrink: 0;
	}

	.profile-avatar-placeholder-small {
		width: 60px;
		height: 60px;
		border-radius: 50%;
		display: flex;
		align-items: center;
		justify-content: center;
		background: var(--efabi-border);
		font-size: 1.5em;
		flex-shrink: 0;
	}

	.profile-info-small {
		flex: 1;
		min-width: 0;
	}

	.profile-name-small {
		margin-bottom: 5px;
		font-size: 1.1em;
	}

	.profile-nip05-small {
		color: var(--efabi-link);
		font-size: 0.8em;
		font-weight: 500;
		word-break: break-all;
	}

	.profile-website-small {
		word-break: break-all;
		color: var(--efabi-link);
	}



	.loading {
		text-align: center;
		padding: 50px;
		color: var(--efabi-text);
		font-size: 1.2em;
		background: white;
		border-radius: 12px;
		box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
	}

	@media (max-width: 768px) {
		.container {
			padding: 12px;
		}

		.carousel-container {
			padding: 20px;
		}

		.carousel-header {
			flex-direction: column;
			gap: 15px;
			align-items: flex-start;
		}

		.carousel-title {
			font-size: 1.6em;
		}

		.carousel-controls {
			align-self: center;
		}

		.carousel-btn {
			width: 45px;
			height: 45px;
			font-size: 20px;
		}

		.card {
			flex: 0 0 280px;
		}

		.card-content {
			padding: 15px;
		}

		.card-title {
			font-size: 1.2em;
		}

		.card-image {
			height: 160px;
		}
	}

	@media (max-width: 480px) {
		.container {
			padding: 8px;
		}

		.carousel-container {
			padding: 15px;
		}

		.carousel-title {
			font-size: 1.4em;
		}

		.carousel-btn {
			width: 40px;
			height: 40px;
			font-size: 18px;
		}

		.card {
			flex: 0 0 250px;
		}

		.card-content {
			padding: 12px;
		}

		.card-title {
			font-size: 1.1em;
		}

		.card-summary {
			font-size: 0.9em;
		}

		.card-image {
			height: 140px;
		}

		.profile-header-small {
			gap: 10px;
		}

		.profile-avatar-small,
		.profile-avatar-placeholder-small {
			width: 50px;
			height: 50px;
		}

		.profile-avatar-placeholder-small {
			font-size: 1.2em;
		}
	}
</style>