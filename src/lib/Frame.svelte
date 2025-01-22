<script>
	import Icon from '@iconify/svelte';

	export let verticalMenuItems = [];
	export let topMenuActions = [];

	export let width = '500px';
	export let height = '500px';
	let activeMenuIndex = null;

	const handleClick = (action) => (action ? action() : console.warn('No action provided'));
	const toggleMenu = (index) => {
		activeMenuIndex = activeMenuIndex === index ? null : index;
	};
</script>

<!-- Layout Container -->
<div class="container" style="height: calc({height} + 40px);">
	<!-- Vertical Icon Menu -->
	<div class="icon-menu">
		{#each verticalMenuItems as item, i}
			<div class="menu-item">
				<button
					class="icon-button"
					on:click={() => {
						if (item.hasSubmenu) {
							toggleMenu(i);
						} else {
							handleClick(item.action);
						}
					}}
					title={item.name}
				>
					<Icon icon={item.icon} style="color: white; width: 24px; height: 24px;" />
				</button>
			</div>
		{/each}
	</div>

	<!-- Submenu as a Vertical Menu Next to the Icon Menu -->
	{#each verticalMenuItems as item, i}
		{#if item.hasSubmenu && activeMenuIndex === i}
			<div class="submenu-container">
				<div class="submenu-menu">
					{#each item.submenuItems as sItem}
						<div class="submenu-item">
							<button
								class="submenu-button"
								on:click={() => handleClick(sItem.action)}
								title={sItem.name}
							>
								<Icon icon={sItem.icon} style="color: white; width: 20px; height: 20px;" />
							</button>
						</div>
					{/each}
				</div>
			</div>
		{/if}
	{/each}

	<!-- Main Canvas Area -->
	<div class="canvas-container">
		<!-- Top Menu -->
		<div class="top-menu" style="width: {width}">
			<!-- Dynamically render top menu actions by filtering for show() === true -->
			{#each topMenuActions as tItem}
				{#if typeof tItem.show === 'function' ? tItem.show() : tItem.show}
					<button
						class="top-menu-button icon-button"
						on:click={() => handleClick(tItem.action)}
						title={typeof tItem.name === 'function' ? tItem.name() : tItem.name}
					>
						{#if tItem.icon}
							<Icon icon={tItem.icon} style="color: white; width: 20px; height: 20px;" />
						{/if}
					</button>
				{/if}
			{/each}
		</div>
		<div class="canvas" style="width: {width}; height: {height};">
			<slot />
		</div>
	</div>
</div>

<style>
	.canvas {
		background-image: linear-gradient(to right, #aeaeae 1px, transparent 1px),
			linear-gradient(to bottom, #aeaeae 1px, transparent 1px);
		background-size: 8.5% 8.5%;
	}
	.container {
		display: flex;
		box-sizing: border-box;
		position: relative;
	}

	.icon-menu {
		width: 60px;
		background-color: #34495e;
		display: flex;
		flex-direction: column;
		align-items: center;
		position: relative;
	}

	.menu-item {
		position: relative;
	}

	.icon-button {
		background: none;
		border: none;
		padding: 10px;
		cursor: pointer;
		transition: background-color 0.3s;
	}

	.icon-button:hover {
		background-color: #2c3e50;
	}

	.icon-button:active {
		background-color: #1abc9c;
	}

	.submenu-container {
		display: flex;
		flex-direction: column;
		background-color: #2c3e50;
		width: 60px;
	}

	.submenu-menu {
		display: flex;
		flex-direction: column;
		align-items: center;
		background-color: #34495e;
	}

	.submenu-item {
		margin: 5px 0;
	}

	.submenu-button {
		background: none;
		border: none;
		padding: 8px;
		cursor: pointer;
		border-radius: 4px;
		transition: background-color 0.3s;
	}

	.submenu-button:hover {
		background-color: #2c3e50;
	}

	.submenu-button:active {
		background-color: #1abc9c;
	}

	.canvas-container {
		flex-grow: 1;
		display: flex;
		flex-direction: column;
		position: relative;
	}

	.top-menu {
		display: flex;
		gap: 10px;
		background-color: #34495e;
		width: 100%;
		height: 40px;
		align-items: center;
	}
</style>
