<script lang="ts">
	import '../app.css';

	let { children } = $props();
</script>

<div class="min-h-screen rounded-lg bg-gray-100 p-4 shadow-md">
	<div class="rounded-lg bg-white p-4 shadow-lg">
		{@render children()}
	</div>
</div>
