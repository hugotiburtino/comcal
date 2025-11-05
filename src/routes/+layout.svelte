<script>
	import { nip19 } from 'nostr-tools';
	import { SimpleSigner } from 'applesauce-signers';
	import { SimpleAccount } from 'applesauce-accounts/accounts';
	import '../app.css';
	import favicon from '$lib/assets/favicon.svg';
	import Navbar from '$lib/components/Navbar.svelte';
	import ModalManager from '$lib/components/ModalManager.svelte';
	import { onMount } from 'svelte';
	import { UserManager, WebStorageStateStore } from 'oidc-client-ts';
	import { manager } from '$lib/stores/accounts.svelte';

	let { children } = $props();

	let errorMessage = $state('');

	async function handleLoginWithPrivateKey(userProfile) {
		let privateKey;
		
		try {
			const nsec = userProfile.nsec;

			if (!userProfile.nsec) {
				throw new Error('No private key found in user profile');
			}

			console.log({nsec});
		
			// Try to decode as regular nsec
			const decoded = nip19.decode(nsec);
			if (decoded.type !== 'nsec') {
				errorMessage = 'Invalid key format. Please enter a valid nsec or ncryptsec key.';
				return;
			}
			privateKey = decoded.data;
			
			// Create signer and account with the private key
			const simpleSigner = new SimpleSigner(privateKey);
			const pk = await simpleSigner.getPublicKey();
			console.log('Public Key:', pk);
			const account = new SimpleAccount(pk, simpleSigner);

			if (!manager.getAccountForPubkey(pk)) {
				manager.addAccount(account);
				manager.setActive(account);
				console.log('LoginWithPrivateKey: New account created and set active:', account);
			} else {
				manager.setActive(account);
				console.log('LoginWithPrivateKey: Existing account set active:', account);
			}

			console.log('LoginWithPrivateKey: Current active account:', manager.active);
			console.log('LoginWithPrivateKey: All accounts:', manager.accounts);
		} catch (error) {
			console.error('Error logging in with private key:', error);
			errorMessage = 'Failed to log in. Please check your key and try again.';
		}
	}

	onMount(async () => {
		const currentUrl = window.location.href;
		
		const urlParams = new URLSearchParams(currentUrl);
		const hasOidcParams = urlParams.has('code') && urlParams.has('session_state');
		// Only process OIDC callback if URL contains 'code' and 'state' parameters
		if (hasOidcParams) {
			const oidcUserManager = new UserManager({
				authority: 'http://localhost:8080/realms/master',
				client_id: 'kanban-board',
				redirect_uri: currentUrl,
				post_logout_redirect_uri: currentUrl,
				response_type: 'code',
				scope: 'openid profile email',
				automaticSilentRenew: true,
				userStore: new WebStorageStateStore({ store: localStorage })
			});
			oidcUserManager.signinCallback().then(user => {
				console.log({user})
				if (user) {
					handleLoginWithPrivateKey(user.profile);
				}
			}).catch(err => {
				if (err?.message !== 'No state in response') {
				console.error('OIDC callback failed:', err);
				}
			});
		}
	});
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
</svelte:head>

<Navbar />
{#if errorMessage}
	<div class="alert alert-danger" role="alert">
		{errorMessage}
	</div>
{/if}
<ModalManager />
{@render children?.()}

