<script lang="ts">
	import { Droplet, Flag, Mountain, BrickWall, TreePine, Bike, AlertTriangle } from 'lucide-svelte';
	
	// Unit Toggle State
	let showPsi = $state(false);
	
	// Force dark mode
	$effect(() => {
		document.documentElement.classList.add('dark');
	});
	
	// Tire models with base pressure values (bar)
	const TIRE_MODELS = [
		{ id: 'continental-5000s-tr', name: 'Continental GP5000 S TR', basePressure: { front: 5.5, rear: 5.5 }, type: 'road', tubeless: false },
		{ id: 'continental-5000', name: 'Continental GP5000', basePressure: { front: 5.5, rear: 5.5 }, type: 'road', tubeless: false },
		{ id: 'schwalbe-pro-one-tt', name: 'Schwalbe Pro One TT', basePressure: { front: 5.8, rear: 5.8 }, type: 'road', tubeless: false },
		{ id: 'pirelli-p-zero-smart', name: 'Pirelli P Zero Smart', basePressure: { front: 5.5, rear: 5.5 }, type: 'road', tubeless: false },
		{ id: 'vittoria-corsa-pro', name: 'Vittoria Corsa Pro', basePressure: { front: 5.8, rear: 5.8 }, type: 'road', tubeless: false },
		{ id: 'michelin-power-time', name: 'Michelin Power Time', basePressure: { front: 5.5, rear: 5.5 }, type: 'road', tubeless: false },
		{ id: 'continental-gatorskin', name: 'Continental Gatorskin', basePressure: { front: 6.0, rear: 6.0 }, type: 'touring', tubeless: false },
		{ id: 'schwalbe-marathon', name: 'Schwalbe Marathon', basePressure: { front: 4.5, rear: 5.0 }, type: 'touring', tubeless: false },
		{ id: 'schwalbe-g-one-bite', name: 'Schwalbe G-One Bite', basePressure: { front: 3.0, rear: 3.2 }, type: 'gravel', tubeless: false },
		{ id: 'continental-terra-speed', name: 'Continental Terra Speed', basePressure: { front: 3.5, rear: 3.8 }, type: 'gravel', tubeless: false },
		{ id: 'schwalbe-racing-ray', name: 'Schwalbe Racing Ray', basePressure: { front: 2.2, rear: 2.4 }, type: 'mtb', tubeless: false },
		{ id: 'continental-5000s-tr-tl', name: 'Continental GP5000 S TR TL', basePressure: { front: 5.2, rear: 5.2 }, type: 'road', tubeless: true },
		{ id: 'schwalbe-g-one-bite-tl', name: 'Schwalbe G-One Bite TL', basePressure: { front: 2.8, rear: 3.0 }, type: 'gravel', tubeless: true },
		{ id: 'schwalbe-nobby-nic-tl', name: 'Schwalbe Nobby Nic TL', basePressure: { front: 1.8, rear: 2.0 }, type: 'mtb', tubeless: true },
		{ id: 'generic-road', name: 'Generic Road', basePressure: { front: 5.5, rear: 5.5 }, type: 'road', tubeless: false },
		{ id: 'generic-gravel', name: 'Generic Gravel', basePressure: { front: 3.5, rear: 3.8 }, type: 'gravel', tubeless: false },
		{ id: 'generic-mtb', name: 'Generic MTB', basePressure: { front: 2.2, rear: 2.4 }, type: 'mtb', tubeless: false }
	];

	const SURFACES = [
		{ id: 'road', name: 'Road', factor: 1.0 },
		{ id: 'wet', name: 'Wet', factor: 0.9 },
		{ id: 'race-track', name: 'Race track', factor: 1.1 },
		{ id: 'gravel', name: 'Gravel', factor: 0.75 },
		{ id: 'cobblestone', name: 'Cobblestone', factor: 0.85 },
		{ id: 'forest', name: 'Forest', factor: 0.8 },
		{ id: 'mud', name: 'Mud', factor: 0.65 }
	];

	const WEIGHT_RANGES = [
		{ min: 0, max: 60, factor: 0.9 },
		{ min: 61, max: 75, factor: 1.0 },
		{ min: 76, max: 90, factor: 1.1 },
		{ min: 91, max: 110, factor: 1.2 },
		{ min: 111, max: 200, factor: 1.3 }
	];

	let weight = $state(75);
	let selectedTire = $state(TIRE_MODELS[0]);
	let selectedSurface = $state(SURFACES[0]);
	let isRearTire = $state(false);
	let customStandardPressure = $state(5.5);
	let isTubeless = $state(false);
	const TUBELESS_FACTOR = 0.93;

	function getWeightFactor(w: number) {
		const range = WEIGHT_RANGES.find(r => w >= r.min && w <= r.max);
		return range ? range.factor : 1.0;
	}

	function getSurfaceButtonClass(surfaceId: string) {
		return selectedSurface.id === surfaceId ? 'bg-blue-500 text-white shadow-lg ring-2 ring-blue-400' : 'bg-gray-700 hover:bg-gray-600 text-gray-300 border border-gray-600';
	}

	// Toggle functions
	function toggleUnit() {
		showPsi = !showPsi;
		localStorage.setItem('showPsi', String(showPsi));
	}

	// Load saved unit preference on mount
	$effect(() => {
		const savedShowPsi = localStorage.getItem('showPsi');
		if (savedShowPsi === 'true') {
			showPsi = true;
		}
	});

	let result = $derived.by(() => {
		const basePressure = isRearTire ? selectedTire.basePressure.rear : selectedTire.basePressure.front;
		const weightFactor = getWeightFactor(weight);
		const surfaceFactor = selectedSurface.factor;
		const tubelessFactor = (selectedTire.tubeless || isTubeless) ? TUBELESS_FACTOR : 1.0;
		const calculatedPressure = basePressure * weightFactor * surfaceFactor * tubelessFactor;
		const resultBar = Math.round(calculatedPressure * 10) / 10;
		const resultPsi = Math.round(calculatedPressure * 14.5038 * 10) / 10;
		const deviation = showPsi 
			? (resultPsi - (customStandardPressure * 14.5038)) 
			: (resultBar - customStandardPressure);
		let warning = '';
		if (Math.abs(deviation) > (showPsi ? 4.35 : 0.3)) { // 0.3 bar ≈ 4.35 psi
			const direction = deviation > 0 ? 'higher' : 'lower';
			const unit = showPsi ? 'psi' : 'bar';
			warning = `${Math.abs(deviation).toFixed(1)} ${unit} ${direction} than standard (${customStandardPressure} bar = ${(customStandardPressure * 14.5038).toFixed(1)} psi)`;
		}
		const isTubelessActive = selectedTire.tubeless || isTubeless;
		const calculationDetails = `Calculation: ${basePressure.toFixed(1)} × ${weightFactor.toFixed(2)} × ${surfaceFactor.toFixed(2)} ${isTubelessActive ? '× 0.93' : ''}`;
		return { resultBar, resultPsi, warning, calculationDetails, isTubelessActive };
	});
	
</script>

<svelte:head>
	<title>TireIQ</title>
	<meta name="description" content="TireIQ - Calculate the optimal tire pressure for your bike" />
	<meta name="viewport" content="width=device-width, initial-scale=1" />
	<meta name="theme-color" content="#1D1D1F" />
	<meta name="apple-mobile-web-app-capable" content="yes" />
	<script>
		// Always apply dark mode
		document.documentElement.classList.add('dark');
		// Easter egg
		console.log(`
  **   ** 
 **** *** 
********* 
 ******* 
  ******  
   ***   
    *    
All Cyclists Are Beautiful`);
	</script>
</svelte:head>

<div class="min-h-screen bg-gray-100 dark:bg-gray-900 p-4 md:p-6 transition-colors duration-300">
	<div class="max-w-2xl mx-auto w-full">
		
		<!-- Header -->
		<div class="text-center mb-6 md:mb-8">
			<h1 class="text-2xl sm:text-3xl md:text-4xl font-bold text-gray-900 dark:text-white mb-2 flex items-center justify-center gap-3">
				<Bike class="w-9 h-9 text-blue-600 dark:text-blue-400 animate-bounce" role="img" aria-hidden="true" style="animation-duration:1s;animation-iteration-count:1" />
				TireIQ
			</h1>
			<p class="text-blue-600 dark:text-blue-400 font-medium text-sm sm:text-base md:text-lg">Be safe. Have fun. Race hard.</p>
			<p class="text-gray-600 dark:text-gray-400 text-sm md:text-base max-w-md mx-auto mt-2">Optimal tire pressure for your bike</p>
			
			<!-- Settings Toggle -->
			<div class="flex justify-center gap-6 mt-6">
				<!-- Unit Toggle -->
				<div class="flex items-center gap-3" role="group" aria-label="Unit">
					<span class="text-sm font-medium text-gray-600 dark:text-gray-400">bar</span>
					<button 
						type="button"
						onclick={toggleUnit}
						class="relative inline-flex items-center h-6 rounded-full w-11 transition-colors duration-200 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2 focus:shadow-[0_0_0_3px_rgba(59,130,246,0.3)] bg-gray-300 dark:bg-gray-600"
						class:bg-blue-500={showPsi}
						aria-label="Toggle between bar and psi"
						aria-pressed={showPsi}
					>
						<span class="absolute left-0.5 top-0.5 bg-white w-5 h-5 rounded-full transition-transform duration-200 shadow-md" class:translate-x-0={!showPsi} class:translate-x-6={showPsi}></span>
					</button>
					<span class="text-sm font-medium text-gray-600 dark:text-gray-400">psi</span>
				</div>
			</div>
		</div>

		<!-- Main Input Card -->
		<div class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 p-5 md:p-6 mb-5 transition-colors">
			<div class="space-y-5">
				
				<!-- Rider Weight -->
				<div class="w-full">
					<label class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-2" for="weight">Rider weight (kg)</label>
					<input 
						id="weight"
						type="number" 
						bind:value={weight} 
						min="40" max="200"
						class="w-full px-3 py-2.5 border border-gray-300 dark:border-gray-600 rounded-lg text-gray-900 dark:text-white bg-white dark:bg-gray-700 focus:ring-2 focus:ring-blue-500 focus:border-blue-500 outline-none transition-colors focus:shadow-[0_0_0_3px_rgba(59,130,246,0.3)]"
					/>
				</div>

				<!-- Tire model -->
				<div class="w-full">
					<label class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-2" for="tire">Tire model</label>
					<select 
						id="tire"
						bind:value={selectedTire}
						class="w-full px-3 py-2.5 border border-gray-300 dark:border-gray-600 rounded-lg bg-white dark:bg-gray-700 text-gray-900 dark:text-white focus:ring-2 focus:ring-blue-500 focus:border-blue-500 outline-none transition-colors focus:shadow-[0_0_0_3px_rgba(59,130,246,0.3)]"
					>
						<optgroup class="text-gray-600" label="Road">
							{#each TIRE_MODELS.filter(t => t.type === 'road') as tire}
								<option value={tire}>{tire.name}{tire.tubeless ? ' (TL)' : ''}</option>
							{/each}
						</optgroup>
						<optgroup class="text-gray-600" label="Touring">
							{#each TIRE_MODELS.filter(t => t.type === 'touring') as tire}
								<option value={tire}>{tire.name}</option>
							{/each}
						</optgroup>
						<optgroup class="text-gray-600" label="Gravel">
							{#each TIRE_MODELS.filter(t => t.type === 'gravel') as tire}
								<option value={tire}>{tire.name}{tire.tubeless ? ' (TL)' : ''}</option>
							{/each}
						</optgroup>
						<optgroup class="text-gray-600" label="MTB">
							{#each TIRE_MODELS.filter(t => t.type === 'mtb') as tire}
								<option value={tire}>{tire.name}{tire.tubeless ? ' (TL)' : ''}</option>
							{/each}
						</optgroup>
					</select>
				</div>

				<!-- Tubeless & Position -->
				<div class="grid grid-cols-2 gap-4">
					<!-- Tubeless Toggle -->
					<div class="flex items-center gap-3" role="group" aria-label="Tire type">
						<span class="text-sm font-medium text-gray-700 dark:text-gray-300">Tube</span>
						<button 
							type="button"
							onclick={() => isTubeless = !isTubeless}
							class="relative inline-flex items-center h-6 rounded-full w-11 transition-colors duration-200 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:shadow-[0_0_0_3px_rgba(59,130,246,0.3)] bg-gray-300 dark:bg-gray-600"
							class:bg-blue-500={isTubeless}
							aria-label="Tubeless mode"
							aria-pressed={isTubeless}
						>
							<span class="absolute left-0.5 top-0.5 bg-white w-5 h-5 rounded-full transition-transform duration-200 shadow-md" class:translate-x-0={!isTubeless} class:translate-x-6={isTubeless}></span>
						</button>
						<span class="text-sm font-medium text-gray-700 dark:text-gray-300">Tubeless</span>
					</div>
					
					<!-- Position Toggle -->
					<div class="flex items-center gap-3" role="group" aria-label="Wheel position">
						<span class="text-sm font-medium text-gray-700 dark:text-gray-300">Front wheel</span>
						<button 
							type="button"
							onclick={() => isRearTire = !isRearTire}
							class="relative inline-flex items-center h-6 rounded-full w-11 transition-colors duration-200 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:shadow-[0_0_0_3px_rgba(59,130,246,0.3)] bg-gray-300 dark:bg-gray-600"
							class:bg-blue-500={isRearTire}
							aria-label="Toggle wheel position"
							aria-pressed={isRearTire}
						>
							<span class="absolute left-0.5 top-0.5 bg-white w-5 h-5 rounded-full transition-transform duration-200 shadow-md" class:translate-x-0={!isRearTire} class:translate-x-6={isRearTire}></span>
						</button>
						<span class="text-sm font-medium text-gray-700 dark:text-gray-300">Rear wheel</span>
					</div>
				</div>

				<!-- Surface Type -->
				<div class="w-full">
					<h2 class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">Surface</h2>
					<div class="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 gap-2" role="radiogroup" aria-label="Select surface">
						{#each SURFACES as surface}
							<button 
								type="button"
								class="px-3 py-2 rounded-lg transition-all {getSurfaceButtonClass(surface.id)} group relative flex flex-col items-center gap-1 min-h-[56px] sm:min-h-[64px] hover:scale-105 duration-200"
								onclick={() => selectedSurface = surface}
								aria-pressed={selectedSurface.id === surface.id}
								role="radio"
							>
								<!-- Icons -->
								{#if surface.id === 'road'}
									<svg class="w-5 h-5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" role="img" aria-hidden="true">
										<path d="M12 2L2 7l10 5 10-5-10-5z"/>
										<path d="M2 17l10 5 10-5"/>
										<path d="M2 12l10 5 10-5"/>
									</svg>
								{:else if surface.id === 'wet'}
									<Droplet class="w-5 h-5" role="img" aria-hidden="true" />
								{:else if surface.id === 'race-track'}
									<Flag class="w-5 h-5" role="img" aria-hidden="true" />
								{:else if surface.id === 'gravel'}
									<Mountain class="w-5 h-5" role="img" aria-hidden="true" />
								{:else if surface.id === 'cobblestone'}
									<BrickWall class="w-5 h-5" role="img" aria-hidden="true" />
								{:else if surface.id === 'forest'}
									<TreePine class="w-5 h-5" role="img" aria-hidden="true" />
								{:else if surface.id === 'mud'}
									<svg class="w-5 h-5" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" role="img" aria-hidden="true">
										<path d="M12 2c-4.97 0-9 4.03-9 9v1c0 4.97 4.03 9 9 9s9-4.03 9-9v-1c0-4.97-4.03-9-9-9z"/>
										<path d="M12 12c-2.21 0-4-1.79-4-4s1.79-4 4-4 4 1.79 4 4-1.79 4-4 4z"/>
									</svg>
								{/if}
								<span class="text-xs text-gray-200 font-medium">{surface.name}</span>
							</button>
						{/each}
					</div>
				</div>

				<!-- Standard pressure -->
				<div class="border-t border-gray-300 dark:border-gray-600 pt-4 w-full">
					<div class="flex items-center gap-2 mb-2 group">
						<label class="block text-sm font-medium text-gray-700 dark:text-gray-300" for="standard">Standard pressure ({showPsi ? 'psi' : 'bar'})</label>
						<button 
							type="button"
							class="relative text-gray-400 dark:text-gray-500 hover:text-blue-500 dark:hover:text-blue-400 transition-colors"
							aria-label="Standard pressure is your usual tire pressure for comparison. Deviations are shown."
						>
							<svg class="w-4 h-4" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" role="img" aria-hidden="true">
								<circle cx="12" cy="12" r="10"/>
								<path d="M12 16v-4M12 8h.01"/>
							</svg>
							<!-- Tooltip -->
							<span class="absolute -top-8 left-1/2 -translate-x-1/2 bg-gray-800 text-white text-xs rounded px-2 py-1 opacity-0 group-hover:opacity-100 transition-opacity duration-150 pointer-events-none whitespace-nowrap z-20 shadow-lg w-max max-w-xs text-center">Standard pressure is your usual tire pressure for comparison. Deviations are shown.</span>
						</button>
					</div>
					<div class="flex gap-3 items-center">
						<input 
							id="standard"
							type="number" bind:value={customStandardPressure}
							step="0.1" min="1.0" max="8.0"
							class="w-full px-3 py-2.5 border border-gray-300 dark:border-gray-600 rounded-lg text-gray-900 dark:text-white bg-white dark:bg-gray-700 focus:ring-2 focus:ring-blue-500 focus:border-blue-500 outline-none transition-colors focus:shadow-[0_0_0_3px_rgba(59,130,246,0.3)]"
						/>
						<span class="text-sm text-gray-500 dark:text-gray-400">±{showPsi ? '4.35' : '0.3'} {showPsi ? 'psi' : 'bar'}</span>
					</div>
				</div>
			</div>
		</div>

		<!-- Result Card -->
		<div class="bg-white dark:bg-gray-800 rounded-2xl shadow-sm border border-gray-200 dark:border-gray-700 p-6 md:p-8 mb-5 transition-colors" role="region" aria-labelledby="result-heading">
			<h2 id="result-heading" class="sr-only">Result</h2>
			<div class="text-center">
				<p class="text-sm text-gray-500 dark:text-gray-400 mb-4">Recommended pressure for <span class="font-medium text-gray-700 dark:text-gray-300">{isRearTire ? 'Rear wheel' : 'Front wheel'}</span></p>
				
				<!-- Main Pressure Display -->
				<div class="relative inline-block mx-auto">
					<div class="absolute inset-0 bg-blue-100 dark:bg-blue-900/30 rounded-3xl blur-xl -z-10"></div>
					<div class="relative bg-gradient-to-b from-blue-50 to-white dark:from-gray-800 dark:to-gray-700 rounded-3xl px-6 py-4">
						<div class="flex justify-center items-baseline gap-2">
							<span class="text-5xl sm:text-6xl md:text-7xl lg:text-8xl xl:text-9xl font-bold text-blue-600 dark:text-blue-400 transition-all duration-300" style="transition-property: transform, opacity;">
								{showPsi ? result.resultPsi.toFixed(1) : result.resultBar.toFixed(1)}
							</span>
							<span class="text-2xl sm:text-2xl sm:text-3xl md:text-4xl lg:text-5xl font-medium text-gray-600 dark:text-gray-300">
								{showPsi ? 'psi' : 'bar'}
							</span>
						</div>
						<p class="text-sm text-gray-500 dark:text-gray-400 mt-2">
							≈ {showPsi ? result.resultBar.toFixed(1) + ' bar' : result.resultPsi.toFixed(1) + ' psi'}
						</p>
					</div>
				</div>
				
				{#if result.warning}
					<div class="bg-amber-50 dark:bg-amber-900/20 border border-amber-200 dark:border-amber-700 rounded-lg p-3 mt-4" role="alert" aria-live="assertive">
						<p class="text-amber-800 dark:text-amber-200 text-sm font-medium flex items-center justify-center gap-2">
							<AlertTriangle class="w-4 h-4" role="img" aria-hidden="true" />
							{result.warning}
						</p>
					</div>
				{/if}

				<!-- Calculation Details -->
				<div class="mt-8 pt-5 border-t border-gray-200 dark:border-gray-700">
					<dl class="grid grid-cols-2 gap-3 text-sm">
						<div class="flex justify-between"><dt class="text-gray-500 dark:text-gray-400">Tire</dt><dd class="text-gray-800 dark:text-gray-200 font-medium">{selectedTire.name}</dd></div>
						<div class="flex justify-between"><dt class="text-gray-500 dark:text-gray-400">Tubeless</dt><dd class="text-gray-800 dark:text-gray-200 font-medium">{isTubeless || selectedTire.tubeless ? 'Yes' : 'No'}</dd></div>
						<div class="flex justify-between"><dt class="text-gray-500 dark:text-gray-400">Surface</dt><dd class="text-gray-800 dark:text-gray-200 font-medium">{selectedSurface.name}</dd></div>
						<div class="flex justify-between"><dt class="text-gray-500 dark:text-gray-400">Position</dt><dd class="text-gray-800 dark:text-gray-200 font-medium">{isRearTire ? 'Rear wheel' : 'Front wheel'}</dd></div>
						<div class="flex justify-between"><dt class="text-gray-500 dark:text-gray-400">Base</dt><dd class="text-gray-800 dark:text-gray-200 font-medium">{isRearTire ? selectedTire.basePressure.rear : selectedTire.basePressure.front} {showPsi ? 'psi' : 'bar'}</dd></div>
						<div class="flex justify-between col-span-2 pt-2 border-t border-gray-200 dark:border-gray-700">
							<dt class="text-gray-500 dark:text-gray-400">Factors</dt>
							<dd class="text-gray-800 dark:text-gray-200 font-medium">{getWeightFactor(weight).toFixed(2)} × {selectedSurface.factor.toFixed(2)}{isTubeless || selectedTire.tubeless ? ' × 0.93' : ''}</dd>
						</div>
					</dl>
					<p class="text-xs text-gray-400 dark:text-gray-500 text-center mt-4" id="calculation-details">{result.calculationDetails}</p>
				</div>
			</div>
		</div>

		<!-- Tips Card -->
		<div class="bg-gray-50 dark:bg-gray-800/80 rounded-2xl border border-gray-200 dark:border-gray-700 p-4 sm:p-5 transition-colors" role="region" aria-labelledby="tips-heading">
			<h3 id="tips-heading" class="font-semibold text-gray-800 dark:text-white mb-3 flex items-center gap-2">
				<svg class="w-5 h-5 text-amber-500" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" role="img" aria-hidden="true">
					<circle cx="12" cy="12" r="10"/>
					<path d="M12 16v-4M12 8h.01"/>
				</svg>
				Tips
			</h3>
			<ul class="text-sm text-gray-600 dark:text-gray-300 space-y-2 list-disc list-inside" role="list">
				<li role="listitem">Check pressure before every ride</li>
				<li role="listitem">Temperature adjustment: Below 15°C +0.1 {showPsi ? 'psi' : 'bar'}, above 25°C -0.1 {showPsi ? 'psi' : 'bar'}</li>
				{#if !(isTubeless || selectedTire.tubeless)}
					<li role="listitem">Tubeless tires: 0.2-0.3 {showPsi ? 'psi' : 'bar'} lower than shown</li>
				{/if}
			</ul>
			<p class="text-xs text-amber-600 dark:text-amber-400 mt-3">Note: All values are recommendations. Adjust pressure to your personal preferences and riding conditions.</p>
		</div>
		
	</div>
</div>
