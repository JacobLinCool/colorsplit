<script lang="ts">
	import '../app.css';
	import { locales, setLocale, getLocale } from '$lib/paraglide/runtime.js';

	let { children } = $props();

	function switchLanguage(locale: (typeof locales)[number]) {
		setLocale(locale, { reload: true });
	}

	const currentLocale = getLocale();
</script>

<div class="min-h-screen">
	<!-- Language Switcher -->
	<nav class="border-b border-gray-200 bg-white">
		<div class="mx-auto max-w-4xl px-4">
			<div class="flex justify-end py-3">
				<div class="flex items-center space-x-2">
					{#each locales as locale}
						<button
							onclick={() => switchLanguage(locale)}
							class="rounded-md px-3 py-1 text-sm transition-colors {currentLocale === locale
								? 'bg-blue-100 font-medium text-blue-800'
								: 'text-gray-600 hover:bg-gray-100'}"
						>
							{locale === 'en' ? 'English' : '繁體中文'}
						</button>
					{/each}
				</div>
			</div>
		</div>
	</nav>

	{@render children()}
</div>
