<script>
	import { pool } from '$lib/stores/nostr-infrastructure.svelte';
	import { nip19 } from 'nostr-tools';
	import { marked } from 'marked';

	// Configuration für verschiedene Content-Publisher
	const publishers = {
		articles: {
			name: 'FOERBICO Artikel',
			npub: 'npub1tgftg8kptdrxxg0g3sm3hckuglv3j0uu3way4vylc5qyt0f44m0s3gun6e',
			kind: 30023
		},
		events: {
			name: 'relilab Events',
			npub: 'npub12j35qpeve33929kg64etvw9g9rzms4c8g5gnqta58yhjdc6wryfse3phmu',
			kind: 31923
		},
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
	let activeTab = $state('efabi-resources');
	let isLoading = $state(true);
	let isConnected = $state(false);
	let connectedRelays = $state(0);

	// Content arrays
	/** @type {any[]} */
	let articles = [];
	/** @type {any[]} */
	let events = [];
	/** @type {any[]} */
	let efabiResources = [];
	/** @type {any[]} */
	let efabiEvents = [];
	/** @type {any[]} */
	let efabiNetwork = [];

	// JSON toggle states
	/** @type {Record<string, boolean>} */
	let jsonToggleStates = $state({});
	/** @type {Record<string, boolean>} */
	let contentToggleStates = $state({});

    // Load content on component mount
    // onMount(() => {
        loadContent();
    // });

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
	 * @param {any} obj
	 */
	function formatJSON(obj) {
		let json = JSON.stringify(obj, null, 2);
		
		json = json.replace(/"([^"]+)":/g, '<span class="json-key">"$1":</span>');
		json = json.replace(/: "([^"]+)"/g, ': <span class="json-string">"$1"</span>');
		json = json.replace(/: (\d+)/g, ': <span class="json-number">$1</span>');
		json = json.replace(/: (true|false)/g, ': <span class="json-boolean">$1</span>');
		
		return json;
	}

	/**
	 * @param {string} tab
	 */
	function switchTab(tab) {
		activeTab = tab;
	}

	/**
	 * @param {string} eventId
	 */
	function toggleJSON(eventId) {
		jsonToggleStates[eventId] = !jsonToggleStates[eventId];
	}

	/**
	 * @param {string} eventId
	 */
	function toggleContent(eventId) {
		contentToggleStates[eventId] = !contentToggleStates[eventId];
	}

	async function loadContent() {
		try {
			console.log('🔄 Starte Laden der Inhalte...');
			
			isConnected = true;
			connectedRelays = relays.length;
			console.log('🔗 Verbinde zu Relays:', relays);

			// Decode publisher pubkeys
			const { data: articleHex } = nip19.decode(publishers.articles.npub);
			const { data: eventHex } = nip19.decode(publishers.events.npub);
			const { data: efabiHex } = nip19.decode(publishers.efabiResources.npub);

			console.log('📊 FOERBICO Hex:', articleHex);
			console.log('📊 relilab Hex:', eventHex);
			console.log('📊 EFABI Hex:', efabiHex);

			// Load FOERBICO Artikel (NIP-23: Long-form Content)
			const articleFilter = {
				kinds: [publishers.articles.kind],
				authors: [articleHex],
				limit: 50
			};
			console.log('🔍 FOERBICO Artikel-Filter:', articleFilter);
			
			// Load relilab Events (NIP-52: Calendar Events)
			const eventFilter = {
				kinds: [publishers.events.kind],
				authors: [eventHex],
				limit: 50
			};
			console.log('🔍 relilab Event-Filter:', eventFilter);

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
				articlesResult,
				eventsResult,
				efabiResourcesResult,
				efabiEventsResult,
				efabiNetworkResult
			] = await Promise.all([
				loadData(articleFilter),
				loadData(eventFilter),
				loadData(efabiResourceFilter),
				loadData(efabiEventFilter),
				loadData(efabiNetworkFilter)
			]);

			articles = articlesResult || [];
			events = eventsResult || [];
			efabiResources = efabiResourcesResult || [];
			efabiEvents = efabiEventsResult || [];
			efabiNetwork = efabiNetworkResult || [];

			console.log('✅ FOERBICO Artikel geladen:', articles.length);
			console.log('✅ relilab Events geladen:', events.length);
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
    <header>
        <div class="badge-success">
            {#if isConnected}
                ✓ Verbunden ({connectedRelays} Relays)
            {:else}
                Verbinde zu Relays...
            {/if}
        </div>
        
        <div class="tabs">
            <!-- <button class="tab-btn" class:active={activeTab === 'articles'} onclick={() => switchTab('articles')}>📄 FOERBICO Artikel</button> -->
            <button class="tab-btn" class:active={activeTab === 'events'} onclick={() => switchTab('events')}>📅 relilab Events</button>
			<button class="tab-btn" class:active={activeTab === 'efabi-resources'} onclick={() => switchTab('efabi-resources')}>📝 efabi Ressourcen</button>
        </div>

        <div class="tabs">
			<button class="tab-btn" class:active={activeTab === 'efabi-network'} onclick={() => switchTab('efabi-network')}>👤 efabi Netzwerkstatt</button>
            <button class="tab-btn" class:active={activeTab === 'efabi-events'} onclick={() => switchTab('efabi-events')}>📆 efabi Veranstaltungen</button>
        </div>
    </header>

    {#if isLoading}
        <div class="loading">Lade Inhalte...</div>
    {:else}
        <!-- FOERBICO Articles - Not working due to static files-->
        <!-- {#if activeTab === 'articles'}
            <div class="articles">
                {#if articles.length === 0}
                    <div class="no-items">Keine Artikel gefunden.</div>
                {:else}
                    {#each articles.sort((a, b) => b.created_at - a.created_at) as article}
                        {@const title = article.tags.find((/** @type {any} */ t) => t[0] === 'title')?.[1] || 'Ohne Titel'}
                        {@const summary = article.tags.find((/** @type {any} */ t) => t[0] === 'summary')?.[1] || ''}
                        {@const image = article.tags.find((/** @type {any} */ t) => t[0] === 'image')?.[1] || ''}
                        {@const publishedAt = article.tags.find((/** @type {any} */ t) => t[0] === 'published_at')?.[1]}
                        {@const tags = article.tags.filter((/** @type {any} */ t) => t[0] === 't').map((/** @type {any} */ t) => t[1])}
                        {@const fullHtmlContent = marked.parse(article.content)}
                        {@const eventId = article.id.substring(0, 8)}
                        {@const displayDate = publishedAt ? parseInt(publishedAt) : article.created_at}
                        
                        <div class="article">
                            <h2 class="article-title">{title}</h2>
                            <div class="article-meta">Veröffentlicht am {formatDate(displayDate)}</div>
                            
                            {#if image}
                                <img src={image} alt={title} class="article-image" />
                            {/if}
                            
                            {#if summary}
                                <div class="article-summary">{summary}</div>
                            {/if}
                            
                            <div class="article-content">{@html fullHtmlContent}</div>
                            
                            {#if tags.length > 0}
                                <div class="tags">
                                    {#each tags as tag}
                                        <span class="tag">{tag}</span>
                                    {/each}
                                </div>
                            {/if}
                            
                            <button class="json-toggle" onclick={() => toggleJSON(eventId)}>
                                {jsonToggleStates[eventId] ? '🔼 JSON ausblenden' : '🔽 JSON anzeigen'}
                            </button>
                            
                            {#if jsonToggleStates[eventId]}
                                <div class="json-container">{@html formatJSON(article)}</div>
                            {/if}
                        </div>
                    {/each}
                {/if}
            </div>
        {/if} -->

        <!-- relilab Events -->
        {#if activeTab === 'events'}
            <div class="events">
                {#if events.length === 0}
                    <div class="no-items">Keine Kalender Events gefunden.</div>
                {:else}
                    {#each events.sort((a, b) => {
                        const aStart = a.tags.find((/** @type {any} */ t) => t[0] === 'start')?.[1] || a.created_at;
                        const bStart = b.tags.find((/** @type {any} */ t) => t[0] === 'start')?.[1] || b.created_at;
                        return parseInt(aStart) - parseInt(bStart);
                    }) as event}
                        {@const title = event.tags.find((/** @type {any} */ t) => t[0] === 'title')?.[1] || 'Ohne Titel'}
                        {@const summary = event.tags.find((/** @type {any} */ t) => t[0] === 'summary')?.[1] || ''}
                        {@const description = event.tags.find((/** @type {any} */ t) => t[0] === 'description')?.[1] || event.content || ''}
                        {@const location = event.tags.find((/** @type {any} */ t) => t[0] === 'location')?.[1] || ''}
                        {@const startTime = event.tags.find((/** @type {any} */ t) => t[0] === 'start')?.[1]}
                        {@const endTime = event.tags.find((/** @type {any} */ t) => t[0] === 'end')?.[1]}
                        {@const image = event.tags.find((/** @type {any} */ t) => t[0] === 'image')?.[1] || ''}
                        {@const tags = event.tags.filter((/** @type {any} */ t) => t[0] === 't').map((/** @type {any} */ t) => t[1])}
                        {@const eventId = event.id.substring(0, 8)}
                        
                        <div class="event">
                            <h2 class="event-title">{title}</h2>
                            <div class="event-meta">Erstellt am {formatDate(event.created_at)}</div>
                            
                            {#if image}
                                <img src={image} alt={title} class="article-image" />
                            {/if}
                            
                            <div class="event-details">
                                {#if startTime}
                                    <div class="event-detail-row">
                                        <span class="event-detail-label">Start:</span>
                                        <span class="event-detail-value">{formatDateTime(parseInt(startTime))}</span>
                                    </div>
                                {/if}
                                {#if endTime}
                                    <div class="event-detail-row">
                                        <span class="event-detail-label">Ende:</span>
                                        <span class="event-detail-value">{formatDateTime(parseInt(endTime))}</span>
                                    </div>
                                {/if}
                                {#if location}
                                    <div class="event-detail-row">
                                        <span class="event-detail-label">Ort:</span>
                                        <span class="event-detail-value">{location}</span>
                                    </div>
                                {/if}
                                {#if summary}
                                    <div class="event-detail-row">
                                        <span class="event-detail-label">Zusammenfassung:</span>
                                        <span class="event-detail-value">{summary}</span>
                                    </div>
                                {/if}
                            </div>
                            
                            {#if description}
                                <div class="article-content">{@html marked.parse(description)}</div>
                            {/if}
                            
                            {#if tags.length > 0}
                                <div class="tags">
                                    {#each tags as tag}
                                        <span class="tag">{tag}</span>
                                    {/each}
                                </div>
                            {/if}
                            
                            <button class="json-toggle" onclick={() => toggleJSON(eventId)}>
                                {jsonToggleStates[eventId] ? '🔼 JSON ausblenden' : '🔽 JSON anzeigen'}
                            </button>
                            
                            {#if jsonToggleStates[eventId]}
                                <div class="json-container">{@html formatJSON(event)}</div>
                            {/if}
                        </div>
                    {/each}
                {/if}
            </div>
        {/if}

        <!-- EFABI Resources -->
        {#if activeTab === 'efabi-resources'}
            <div class="articles">
                {#if efabiResources.length === 0}
                    <div class="no-items">Keine efabi Ressourcen gefunden.</div>
                {:else}
                    {#each efabiResources.sort((a, b) => b.created_at - a.created_at) as article}
                        {@const title = article.tags.find((/** @type {any} */ t) => t[0] === 'title')?.[1] || 'Ohne Titel'}
                        {@const summary = article.tags.find((/** @type {any} */ t) => t[0] === 'summary')?.[1] || ''}
                        {@const image = article.tags.find((/** @type {any} */ t) => t[0] === 'image')?.[1] || ''}
                        {@const publishedAt = article.tags.find((/** @type {any} */ t) => t[0] === 'published_at')?.[1]}
                        {@const tags = article.tags.filter((/** @type {any} */ t) => t[0] === 't').map((/** @type {any} */ t) => t[1])}
                        {@const fullHtmlContent = marked.parse(article.content)}
                        {@const eventId = article.id.substring(0, 8)}
                        {@const displayDate = publishedAt ? parseInt(publishedAt) : article.created_at}
                        
                        <div class="article">
                            <h2 class="article-title">{title}</h2>
                            <div class="article-meta">Veröffentlicht am {formatDate(displayDate)}</div>
                            
                            {#if image}
                                <img src={image} alt={title} class="article-image" />
                            {/if}
                            
                            {#if summary}
                                <div class="article-summary">{summary}</div>
                            {/if}
                            
                            <div class="article-content">{@html fullHtmlContent}</div>
                            
                            {#if tags.length > 0}
                                <div class="tags">
                                    {#each tags as tag}
                                        <span class="tag">{tag}</span>
                                    {/each}
                                </div>
                            {/if}
                            
                            <button class="json-toggle" onclick={() => toggleJSON(eventId)}>
                                {jsonToggleStates[eventId] ? '🔼 JSON ausblenden' : '🔽 JSON anzeigen'}
                            </button>
                            
                            {#if jsonToggleStates[eventId]}
                                <div class="json-container">{@html formatJSON(article)}</div>
                            {/if}
                        </div>
                    {/each}
                {/if}
            </div>
        {/if}

        <!-- EFABI Events -->
        {#if activeTab === 'efabi-events'}
            <div class="events">
                {#if efabiEvents.length === 0}
                    <div class="no-items">Keine efabi Veranstaltungen gefunden.</div>
                {:else}
                    {#each efabiEvents.sort((a, b) => {
                        const aStart = a.tags.find((/** @type {any} */ t) => t[0] === 'start')?.[1] || a.created_at;
                        const bStart = b.tags.find((/** @type {any} */ t) => t[0] === 'start')?.[1] || b.created_at;
                        return parseInt(aStart) - parseInt(bStart);
                    }) as event}
                        {@const title = event.tags.find((/** @type {any} */ t) => t[0] === 'title')?.[1] || 'Ohne Titel'}
                        {@const summary = event.tags.find((/** @type {any} */ t) => t[0] === 'summary')?.[1] || ''}
                        {@const description = event.tags.find((/** @type {any} */ t) => t[0] === 'description')?.[1] || event.content || ''}
                        {@const location = event.tags.find((/** @type {any} */ t) => t[0] === 'location')?.[1] || ''}
                        {@const startTime = event.tags.find((/** @type {any} */ t) => t[0] === 'start')?.[1]}
                        {@const endTime = event.tags.find((/** @type {any} */ t) => t[0] === 'end')?.[1]}
                        {@const image = event.tags.find((/** @type {any} */ t) => t[0] === 'image')?.[1] || ''}
                        {@const tags = event.tags.filter((/** @type {any} */ t) => t[0] === 't').map((/** @type {any} */ t) => t[1])}
                        {@const eventId = event.id.substring(0, 8)}
                        
                        <div class="event">
                            <h2 class="event-title">{title}</h2>
                            <div class="event-meta">Erstellt am {formatDate(event.created_at)}</div>
                            
                            {#if image}
                                <img src={image} alt={title} class="article-image" />
                            {/if}
                            
                            <div class="event-details">
                                {#if startTime}
                                    <div class="event-detail-row">
                                        <span class="event-detail-label">Start:</span>
                                        <span class="event-detail-value">{formatDateTime(parseInt(startTime))}</span>
                                    </div>
                                {/if}
                                {#if endTime}
                                    <div class="event-detail-row">
                                        <span class="event-detail-label">Ende:</span>
                                        <span class="event-detail-value">{formatDateTime(parseInt(endTime))}</span>
                                    </div>
                                {/if}
                                {#if location}
                                    <div class="event-detail-row">
                                        <span class="event-detail-label">Ort:</span>
                                        <span class="event-detail-value">{location}</span>
                                    </div>
                                {/if}
                                {#if summary}
                                    <div class="event-detail-row">
                                        <span class="event-detail-label">Zusammenfassung:</span>
                                        <span class="event-detail-value">{summary}</span>
                                    </div>
                                {/if}
                            </div>
                            
                            {#if description}
                                <div class="article-content">{@html marked.parse(description)}</div>
                            {/if}
                            
                            {#if tags.length > 0}
                                <div class="tags">
                                    {#each tags as tag}
                                        <span class="tag">{tag}</span>
                                    {/each}
                                </div>
                            {/if}
                            
                            <button class="json-toggle" onclick={() => toggleJSON(eventId)}>
                                {jsonToggleStates[eventId] ? '🔼 JSON ausblenden' : '🔽 JSON anzeigen'}
                            </button>
                            
                            {#if jsonToggleStates[eventId]}
                                <div class="json-container">{@html formatJSON(event)}</div>
                            {/if}
                        </div>
                    {/each}
                {/if}
            </div>
        {/if}

        <!-- EFABI Network -->
        {#if activeTab === 'efabi-network'}
            <div class="network">
                {#if efabiNetwork.length === 0}
                    <div class="no-items">Keine Netzwerkstatt-Profile gefunden.</div>
                {:else}
                    {#each efabiNetwork as profile}
                        {@const profileData = JSON.parse(profile.content || '{}')}
                        {@const eventId = profile.id.substring(0, 8)}
                        
                        <div class="network-profile">
                            <div class="profile-header">
                                {#if profileData.picture}
                                    <img src={profileData.picture} alt={profileData.name || 'Profil'} class="profile-avatar" />
                                {:else}
                                    <div class="profile-avatar-placeholder">👤</div>
                                {/if}
                                
                                <div class="profile-info">
                                    {#if profileData.name}
                                        <h3 class="profile-name">{profileData.name}</h3>
                                    {/if}
                                    
                                    {#if profileData.nip05}
                                        <div class="profile-nip05">{profileData.nip05}</div>
                                    {/if}
                                    
                                    {#if profileData.about}
                                        <div class="profile-about">
                                            {@html marked.parse(profileData.about)}
                                        </div>
                                    {/if}
                                    
                                    {#if profileData.website}
                                        <div class="profile-website">
                                            <a href={profileData.website} target="_blank" rel="noopener noreferrer">
                                                {profileData.website}
                                            </a>
                                        </div>
                                    {/if}
                                </div>
                            </div>
                            
                            <button class="json-toggle" onclick={() => toggleJSON(eventId)}>
                                {jsonToggleStates[eventId] ? '🔼 JSON ausblenden' : '🔽 JSON anzeigen'}
                            </button>
                            
                            {#if jsonToggleStates[eventId]}
                                <div class="json-container">{@html formatJSON(profile)}</div>
                            {/if}
                        </div>
                    {/each}
                {/if}
            </div>
        {/if}
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
		max-width: 1000px;
		margin: 0 auto;
		padding: 20px;
	}

	header {
		background: white;
		padding: 30px 40px;
		border-radius: 12px;
		box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
		margin-bottom: 30px;
		text-align: center;
		border-bottom: 4px solid var(--efabi-h1);
	}

	.logo-container {
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 15px;
		margin-bottom: 15px;
	}

	.logo-container img {
		height: 60px;
		width: auto;
	}

	h1 {
		color: var(--efabi-h2);
		font-size: 2.2em;
		margin-bottom: 8px;
		font-weight: 700;
		display: inline-block;
	}

	.subtitle {
		color: var(--efabi-h2);
		font-size: 1.1em;
		margin-bottom: 25px;
		opacity: 0.9;
	}

	.tabs {
		display: flex;
		gap: 12px;
		margin-top: 25px;
		justify-content: center;
		flex-wrap: wrap;
	}

	.tab-btn {
		padding: 12px 28px;
		background: var(--efabi-border);
		border: 2px solid transparent;
		border-radius: 8px;
		cursor: pointer;
		font-size: 1em;
		transition: all 0.3s ease;
		color: var(--efabi-text);
		font-weight: 500;
	}

	.tab-btn.active {
		background: var(--efabi-link);
		color: white;
		border-color: var(--efabi-link-hover);
		box-shadow: 0 4px 12px rgba(63, 64, 247, 0.2);
	}

	.tab-btn:hover {
		background: #e0e0e0;
	}

	.tab-btn.active:hover {
		background: var(--efabi-link-hover);
	}

	.status {
		display: inline-block;
		padding: 10px 20px;
		border-radius: 20px;
		font-size: 0.95em;
		margin-top: 15px;
		font-weight: 500;
	}

	.status.connecting {
		background: #fff3cd;
		color: var(--efabi-h2);
	}

	.status.connected {
		background: #d4edda;
		color: #155724;
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

	.articles, .events {
		display: grid;
		gap: 28px;
	}

	.article, .event {
		background: white;
		padding: 32px;
		border-radius: 12px;
		box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
		transition: transform 0.3s ease, box-shadow 0.3s ease;
		border-left: 4px solid var(--efabi-border);
	}

	.article:hover, .event:hover {
		transform: translateY(-3px);
		box-shadow: 0 8px 20px rgba(0, 0, 0, 0.12);
		border-left-color: var(--efabi-link);
	}

	.article-title, .event-title {
		font-size: 1.9em;
		color: var(--efabi-h2);
		margin-bottom: 12px;
		line-height: 1.3;
		font-weight: 700;
	}

	.article-meta, .event-meta {
		color: var(--efabi-text);
		font-size: 0.95em;
		margin-bottom: 20px;
		padding-bottom: 16px;
		border-bottom: 2px solid var(--efabi-border);
		opacity: 0.85;
	}

	.event-details {
		background: #f8f9fa;
		padding: 18px;
		border-radius: 8px;
		margin: 18px 0;
		border-left: 4px solid var(--efabi-link);
	}

	.event-detail-row {
		margin-bottom: 12px;
		display: flex;
		align-items: start;
		gap: 12px;
	}

	.event-detail-row:last-child {
		margin-bottom: 0;
	}

	.event-detail-label {
		font-weight: 600;
		color: var(--efabi-h2);
		min-width: 100px;
	}

	.event-detail-value {
		color: var(--efabi-text);
		word-break: break-word;
		flex: 1;
	}

	.article-summary {
		color: var(--efabi-text);
		font-size: 1.08em;
		line-height: 1.7;
		margin-bottom: 18px;
		font-style: italic;
		background: #f8f9fa;
		padding: 16px;
		border-left: 3px solid var(--efabi-h1);
		border-radius: 4px;
	}

	.article-content {
		color: var(--efabi-text);
		line-height: 1.9;
		font-size: 1.05em;
	}

	.article-content :global(h1),
	.article-content :global(h2),
	.article-content :global(h3),
	.article-content :global(h4) {
		margin-top: 24px;
		margin-bottom: 12px;
		color: var(--efabi-h2);
		font-weight: 700;
	}

	.article-content :global(h1) {
		font-size: 1.9em;
		border-bottom: 2px solid var(--efabi-border);
		padding-bottom: 12px;
	}

	.article-content :global(h2) {
		font-size: 1.6em;
	}

	.article-content :global(h3) {
		font-size: 1.4em;
	}

	.article-content :global(p) {
		margin-bottom: 16px;
	}

	.article-content :global(a) {
		color: var(--efabi-link);
		text-decoration: none;
		border-bottom: 1px solid var(--efabi-link);
		transition: color 0.2s ease;
	}

	.article-content :global(a:hover) {
		color: var(--efabi-link-hover);
		border-bottom-color: var(--efabi-link-hover);
	}

	.article-content :global(img) {
		max-width: 100%;
		height: auto;
		border-radius: 8px;
		margin: 20px 0;
		display: block;
	}

	.article-content :global(ul),
	.article-content :global(ol) {
		margin-left: 20px;
		margin-bottom: 15px;
	}

	.article-content :global(li) {
		margin-bottom: 8px;
	}

	.article-content :global(blockquote) {
		border-left: 4px solid #667eea;
		padding-left: 20px;
		margin: 20px 0;
		color: #636e72;
		font-style: italic;
	}

	.article-content :global(code) {
		background: #f0f0f0;
		padding: 2px 6px;
		border-radius: 4px;
		font-family: 'Courier New', monospace;
		font-size: 0.9em;
	}

	.article-content :global(pre) {
		background: #2d3436;
		color: #55efc4;
		padding: 15px;
		border-radius: 8px;
		overflow-x: auto;
		margin: 15px 0;
	}

	.article-content :global(pre code) {
		background: none;
		padding: 0;
		color: inherit;
	}

	.tags {
		display: flex;
		flex-wrap: wrap;
		gap: 10px;
		margin-top: 22px;
	}

	.tag {
		background: #e8f1ff;
		color: var(--efabi-link);
		padding: 6px 14px;
		border-radius: 16px;
		font-size: 0.85em;
		font-weight: 500;
		border: 1px solid var(--efabi-border);
	}

	.no-items {
		background: white;
		padding: 50px;
		border-radius: 12px;
		text-align: center;
		color: var(--efabi-text);
		box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
	}

	.json-toggle {
		margin-top: 22px;
		padding: 11px 22px;
		background: var(--efabi-border);
		border: 1px solid #d0d8e0;
		border-radius: 8px;
		cursor: pointer;
		font-size: 0.9em;
		color: var(--efabi-text);
		transition: all 0.2s ease;
		font-weight: 500;
	}

	.json-toggle:hover {
		background: #d0d8e0;
	}

	.article-image {
		width: 100%;
		max-width: 100%;
		height: auto;
		border-radius: 12px;
		margin: 20px 0;
		object-fit: cover;
	}

	.json-container {
		margin-top: 15px;
		background: #2d3436;
		color: #55efc4;
		padding: 20px;
		border-radius: 8px;
		overflow-x: auto;
		font-family: 'Courier New', monospace;
		font-size: 0.85em;
		line-height: 1.5;
		max-height: 500px;
		overflow-y: auto;
		word-wrap: break-word;
		overflow-wrap: break-word;
		white-space: pre-wrap;
		max-width: 100%;
	}

	.json-container :global(.json-key) {
		color: #74b9ff;
		word-break: break-all;
	}

	.json-container :global(.json-string) {
		color: #55efc4;
		word-break: break-all;
	}

	.json-container :global(.json-number) {
		color: #fdcb6e;
	}

	.json-container :global(.json-boolean) {
		color: #ff7675;
	}

	.network {
		display: flex;
		flex-direction: column;
		gap: 20px;
	}

	.network-profile {
		background: white;
		padding: 25px;
		border-radius: 12px;
		box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
		border-left: 4px solid var(--efabi-border);
		transition: border-left-color 0.3s ease;
	}

	.network-profile:hover {
		border-left-color: var(--efabi-link);
	}

	.profile-header {
		display: flex;
		gap: 20px;
		align-items: flex-start;
		margin-bottom: 20px;
	}

	.profile-avatar {
		width: 80px;
		height: 80px;
		border-radius: 50%;
		object-fit: cover;
		border: 3px solid var(--efabi-link);
	}

	.profile-avatar-placeholder {
		width: 80px;
		height: 80px;
		border-radius: 50%;
		display: flex;
		align-items: center;
		justify-content: center;
		background: var(--efabi-border);
		font-size: 2em;
		flex-shrink: 0;
	}

	.profile-info {
		flex: 1;
	}

	.profile-name {
		color: var(--efabi-h2);
		font-size: 1.5em;
		margin-bottom: 5px;
		font-weight: 700;
	}

	.profile-nip05 {
		color: var(--efabi-link);
		font-size: 0.95em;
		font-weight: 500;
	}

	.profile-about {
		color: var(--efabi-text);
		margin-bottom: 15px;
		line-height: 1.6;
	}

	.profile-about :global(p) {
		margin-bottom: 10px;
	}

	.profile-website {
		margin-bottom: 15px;
	}

	.profile-website :global(a) {
		color: var(--efabi-link);
		text-decoration: none;
		border-bottom: 1px solid var(--efabi-link);
		transition: color 0.2s ease;
	}

	.profile-website :global(a:hover) {
		color: var(--efabi-link-hover);
		border-bottom-color: var(--efabi-link-hover);
	}

	@media (max-width: 768px) {
		.container {
			padding: 12px;
		}

		h1 {
			font-size: 2.2em;
		}

		.subtitle {
			font-size: 1em;
		}

		header {
			padding: 25px;
		}

		.article, .event {
			padding: 22px;
		}

		.article-title, .event-title {
			font-size: 1.5em;
		}

		.article-content {
			font-size: 1em;
		}

		.json-container {
			font-size: 0.75em;
			padding: 15px;
			max-height: 300px;
		}

		.json-toggle {
			width: 100%;
			padding: 12px;
		}

		.tags {
			gap: 6px;
		}

		.tag {
			font-size: 0.8em;
		}

		.tabs {
			gap: 8px;
		}

		.tab-btn {
			padding: 10px 16px;
			font-size: 0.9em;
		}
	}

	@media (max-width: 480px) {
		h1 {
			font-size: 1.8em;
		}

		.subtitle {
			font-size: 0.95em;
		}

		header {
			padding: 18px;
		}

		.article, .event {
			padding: 16px;
		}

		.article-title, .event-title {
			font-size: 1.3em;
		}

		.article-meta, .event-meta {
			font-size: 0.9em;
		}

		.article-content {
			font-size: 0.95em;
		}

		.json-container {
			font-size: 0.7em;
			padding: 10px;
		}

		.tabs {
			flex-direction: column;
		}

		.tab-btn {
			width: 100%;
		}
	}
</style>