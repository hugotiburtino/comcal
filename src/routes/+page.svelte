<script>
	import { goto } from '$app/navigation';
	import { useActiveUser } from '$lib/stores/accounts.svelte';
	import { useJoinedCommunitiesList } from '$lib/stores/joined-communities-list.svelte.js';
	import { getTagValue } from 'applesauce-core/helpers';
	import { hexToNpub } from '$lib/helpers/nostrUtils.js';
	import LandingHero from '$lib/components/landing/LandingHero.svelte';
	import FeatureHighlights from '$lib/components/landing/FeatureHighlights.svelte';
	import CommunityCarousel from '$lib/components/landing/CommunityCarousel.svelte';

	const activeUser = useActiveUser();
	const getJoinedCommunities = useJoinedCommunitiesList();

	// Redirect logged-in users to first community or discover page
	$effect(() => {
		if (activeUser()) {
			const communities = getJoinedCommunities();
			if (communities.length > 0) {
				// Redirect to first community
				const firstCommunity = communities.sort()[0];
				const pubkey = getTagValue(firstCommunity, 'd');
				if (pubkey) {
					const npub = hexToNpub(pubkey);
					if (npub) {
						goto(`/c/${npub}`);
					}
				}
			}
		}
	});
</script>

<svelte:head>
	<title>efabiNet - Neu</title>
	<meta
		name="description"
		content="der Plattform für das bundesweite Netzwerk der Evangelischen Familienbildung"
	/>
	<link rel="icon" href="https://efabi.net/wp-content/uploads/2025/06/cropped-Logo-fuer-Header-32x32.png" sizes="32x32">
</svelte:head>

<!-- {#if !activeUser()} -->
	<!-- Landing page: For non-logged-in users -->
	<LandingHero />
	<!-- <FeatureHighlights /> -->
	<CommunityCarousel />
<!-- {/if} -->
