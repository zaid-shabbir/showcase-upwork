<script lang="ts">
	import { createEventDispatcher } from "svelte";
	import BackIcon from "./icons/BackIcon.svelte";
	import HomeIcon from "./icons/HomeIcon.svelte";

	const dispatch = createEventDispatcher();
	export let currentPath: string;

	// pathParts does not include root
	$: pathParts = currentPath.length == 0 ? [] : currentPath.indexOf('/') !== -1
		  ? currentPath.split('/') 
		  : [currentPath];

	/** level: 0 is root */
	function pathForLevel(level) {
		let path = "";
		for (let i = 0; i < level; i++) {
			if (path.length > 0) path += "/";
			path += pathParts[i];
		}
		return path;
	}

	function navigateTo(e, level) {
		e.preventDefault();
		const path = pathForLevel(level);
		dispatch('navigate', {path});
	}
</script>

<div>
	<h4>
		<span class="dl-back" class:dl-back-disabled={pathParts.length == 0}>
			<a href="#0" alt="Back" on:click={e => navigateTo(e, pathParts.length - 1)}>
				<span class="dl-icon">
					<BackIcon />
				</span>
			</a>
		</span>

		<a href="#0">
			<a href="#0" alt="Home" on:click={e => navigateTo(e, 0)}>
				<span class="dl-icon">
					<HomeIcon />
				</span>
			</a>
		</a>

		{#each pathParts as part, i }
			<span class="dl-seperator">&gt;</span>
			<a href="#dataroom:{pathForLevel(i + 1)}" on:click|preventDefault|stopPropagation={e => navigateTo(e, i + 1)}>{part}</a>
		{/each}
	</h4>
</div>

<style>
	.dl-back {
		margin-right: 2rem;
	}
	.dl-back-disabled {
		opacity: 0.3;
	}

	.dl-icon {
		overflow: auto;
		vertical-align: middle;
		padding: 0.2rem 0 0rem 0;
		opacity: 0.8;
	}

	.dl-seperator {
		margin-left: 0.7rem;
		margin-right: 0.7rem;
	}
</style>