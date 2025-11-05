<script>
	import { UserManager, WebStorageStateStore } from 'oidc-client-ts';

	let { class: className = '' } = $props();

	function initializeOidcFlow(currentUrl) {
		const oidcUserManager = new UserManager({
			authority: 'https://kc.auth.tchncs.de/realms/nostr-onboarding/account',
			client_id: 'efabi-neu',
			redirect_uri: currentUrl,
			post_logout_redirect_uri: currentUrl,
			response_type: 'code',
			scope: 'openid profile email',
			automaticSilentRenew: true,
			userStore: new WebStorageStateStore({ store: localStorage })
		});
		oidcUserManager.signinRedirect();
	}
</script>

<button class="btn btn-secondary {className}" onclick={initializeOidcFlow(window.location.href)}>
	Sign Up!
</button>
