<script>
	import { modalStore } from '$lib/stores/modal.svelte.js';

	function handleGetStarted() {
		modalStore.openModal('signup');
	}

	function scrollToCommunities() {
		const carousel = document.querySelector('#community-showcase');
		if (carousel) {
			carousel.scrollIntoView({ behavior: 'smooth', block: 'start' });
		}
	}
</script>

<section class="relative overflow-hidden bg-base-300 py-20 text-primary-content">
	<div class="container relative z-10 mx-auto px-4">
		<!-- Main headline -->
		<div class="mx-auto max-w-5xl text-center">
			<h1 class="mb-6 text-4xl font-bold leading-tight md:text-5xl lg:text-6xl" style="color: oklch(80% 38% 76deg)">
				Herzlich Willkommen bei efabiNet
			</h1>
			<p class="mb-8 text-lg opacity-90 md:text-xl lg:text-2xl font-bold">
				der Plattform für das bundesweite Netzwerk der Evangelischen Familienbildung
			</p>

		<!-- Feature badges -->
		<div class="mb-10 flex flex-wrap justify-center gap-4">
			<div class="flex items-center gap-2 rounded-full bg-white/20 px-4 py-2 backdrop-blur-sm">
				<span class="text-2xl">🏘️</span>
				<span class="font-medium">Communities</span>
			</div>
			<div class="flex items-center gap-2 rounded-full bg-white/20 px-4 py-2 backdrop-blur-sm">
				<span class="text-2xl">💬</span>
				<span class="font-medium">Chat</span>
			</div>
			<a 
				href="/calendar"
				class="flex items-center gap-2 rounded-full bg-white/20 px-4 py-2 backdrop-blur-sm transition-opacity hover:opacity-80 cursor-pointer"
			>
				<span class="text-2xl">📅</span>
				<span class="font-medium">Calendar</span>
			</a>
			<div class="flex items-center gap-2 rounded-full bg-white/20 px-4 py-2 backdrop-blur-sm">
				<span class="text-2xl">✨</span>
				<span class="font-medium">More to come</span>
			</div>
		</div>

			<!-- Call to action buttons -->
			<div class="flex flex-col items-center justify-center gap-4 sm:flex-row">
				<a
					href="/calendar"
					class="btn btn-lg bg-white text-primary hover:bg-white/90 border-none shadow-lg"
				>
					Browse Calendar
			</a>
				<a
					href="/discover"
					class="btn btn-lg btn-outline border-2 border-white text-white hover:bg-white hover:text-primary"
				>
					Explore Communities
			</a>
			</div>
		</div>
	</div>
</section>
