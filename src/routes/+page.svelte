<script lang="ts">
	let colors = ['#E18D93', '#8F9F72', '#5A6E48', '#EDCC8A', '#ADC9CC'];
	let newColor = '#000000';
	let uploadedSvg: string | null = null;
	let expectedColorCount = 4;
	let originalColors: string[] = [];
	let isValidColorCount = true;
	let uniqueColorCount = 0;

	function validateUpload(): boolean {
		return expectedColorCount > 0 && expectedColorCount <= colors.length;
	}

	function addColor() {
		colors = [...colors, newColor];
		newColor = '#000000';
	}

	function removeColor(index: number) {
		colors = colors.filter((_, i) => i !== index);
	}

	async function handleFileUpload(event: Event) {
		const target = event.target as HTMLInputElement;
		if (!validateUpload()) {
			alert(`Please set a valid number of expected colors (1-${colors.length})`);
			target.value = ''; // Reset file input
			return;
		}

		const file = target.files?.[0];
		if (file && file.type === 'image/svg+xml') {
			const reader = new FileReader();
			reader.onload = (e) => {
				const content = e.target?.result as string;
				uploadedSvg = content;
				
				const fillRegex = /fill="([^"]*)"/g;
				const matches = content.match(fillRegex);
				
				if (!matches) {
					originalColors = [];
					return;
				}

				originalColors = matches
					.map(match => match.match(/fill="([^"]*)"/)?.[1])
					.filter(color => color && color !== 'none' && 
						(color.startsWith('#') || 
						color.startsWith('rgb') || 
						color.startsWith('hsl'))) as string[];

				normalizeColors();
			};
			reader.readAsText(file);
		}
	}

	function normalizeColors() {
		if (!uploadedSvg || !originalColors.length) return;

		// Function to convert color to RGB values
		function colorToRGB(color: string): [number, number, number] {
			const canvas = document.createElement('canvas');
			const ctx = canvas.getContext('2d');
			if (!ctx) return [0, 0, 0];

			ctx.fillStyle = color;
			const hex = ctx.fillStyle;
			
			return [
				parseInt(hex.slice(1, 3), 16),
				parseInt(hex.slice(3, 5), 16),
				parseInt(hex.slice(5, 7), 16)
			];
		}

		// Function to calculate color distance
		function colorDistance(color1: string, color2: string): number {
			const [r1, g1, b1] = colorToRGB(color1);
			const [r2, g2, b2] = colorToRGB(color2);
			
			return Math.sqrt(
				Math.pow(r1 - r2, 2) +
				Math.pow(g1 - g2, 2) +
				Math.pow(b1 - b2, 2)
			);
		}

		// Group similar colors using k-means-like approach
		const uniqueColors = Array.from(new Set(originalColors));
		const colorGroups: { [key: string]: string[] } = {};
		
		// Initialize groups with the first n colors
		const representatives = uniqueColors.slice(0, expectedColorCount);
		representatives.forEach(color => {
			colorGroups[color] = [];
		});

		// Assign each color to the nearest representative
		uniqueColors.forEach(color => {
			let minDistance = Infinity;
			let nearestRep = representatives[0];
			
			representatives.forEach(rep => {
				const distance = colorDistance(color, rep);
				if (distance < minDistance) {
					minDistance = distance;
					nearestRep = rep;
				}
			});
			
			colorGroups[nearestRep].push(color);
		});

		// Update uniqueColorCount and modify SVG
		uniqueColorCount = expectedColorCount;
		let modifiedSvg = uploadedSvg;
		
		// Replace each color with its representative
		Object.entries(colorGroups).forEach(([representative, colors]) => {
			colors.forEach(color => {
				const regex = new RegExp(`fill="${color}"`, 'g');
				modifiedSvg = modifiedSvg.replace(regex, `fill="${representative}"`);
			});
		});
		
		uploadedSvg = modifiedSvg;
	}

	$: isValidColorCount = expectedColorCount > 0 && expectedColorCount <= colors.length;

	$: combinations = generateCombinations(colors, uniqueColorCount);

	function generateCombinations(colors: string[], n: number) {
		if (n > colors.length) return []; // Not enough colors for the combination
		
		const result = [];
		const combination = new Array(n);
		const used = new Array(colors.length).fill(false);
		
		function generate(position: number) {
			if (position === n) {
				result.push([...combination]);
				return;
			}
			
			for (let i = 0; i < colors.length; i++) {
				if (!used[i]) {
					used[i] = true;
					combination[position] = colors[i];
					generate(position + 1);
					used[i] = false;
				}
			}
		}
		
		generate(0);
		return result;
	}

	function getSvgUrl(colorCombo: string[]) {
		if (!uploadedSvg) {
			// Use default SVG if none uploaded
			const svgContent = `<?xml version="1.0" encoding="UTF-8"?>
				// ... your default SVG content ...
			`;
			return `data:image/svg+xml,${encodeURIComponent(svgContent)}`;
		}

		// Replace colors in the uploaded SVG
		let modifiedSvg = uploadedSvg;
		const originalColors = [...new Set(Array.from(modifiedSvg.matchAll(/fill="([^"]*)"/g)).map(m => m[1]))];
		
		originalColors.forEach((originalColor, index) => {
			if (index < colorCombo.length) {
				const regex = new RegExp(`fill="${originalColor}"`, 'g');
				modifiedSvg = modifiedSvg.replace(regex, `fill="${colorCombo[index]}"`);
			}
		});

		return `data:image/svg+xml,${encodeURIComponent(modifiedSvg)}`;
	}
</script>

<div class="max-w-7xl mx-auto p-6 bg-gray-50 min-h-screen my-20">
	<h1 class="text-3xl font-bold mb-8 text-gray-800 border-b pb-4">SVG Color Pattern Generator</h1>

	<div class="mb-6 bg-blue-50 border border-blue-200 rounded-lg p-4">
		<h2 class="text-lg font-semibold mb-2 text-blue-800">📝 Quick Tutorial</h2>
		<p class="text-blue-700 mb-2">This tool only works with SVG files. If you have a PNG or other image format:</p>
		<ol class="list-decimal list-inside text-blue-700 ml-2">
			<li>Convert your image to SVG using a converter like <a href="https://www.freeconvert.com/png-to-svg" class="text-blue-600 underline hover:text-blue-800" target="_blank" rel="noopener noreferrer">FreeConvert's PNG to SVG tool</a></li>
			<li>Select how many unique colors there is in your art</li>
			<li>Upload the converted SVG file below</li>
			<li>Choose your desired colors!</li>
		</ol>
	</div>

	<div class="mb-12 space-y-6 bg-white p-6 rounded-lg shadow-sm">
		<div>
			<h2 class="text-lg font-semibold mb-4 text-gray-700">Set Expected Colors</h2>
			<div class="flex items-center gap-4">
				<div class="w-64">
					<label class="block text-sm font-medium text-gray-700 mb-1">
						Number of Colors in SVG:
					</label>
					<input
						type="number"
						bind:value={expectedColorCount}
						min="1"
						max={colors.length}
						class="block w-32 rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500"
					/>
				</div>
				{#if !isValidColorCount}
					<p class="text-red-500 text-sm">
						Please enter a number between 1 and {colors.length}
					</p>
				{/if}
			</div>
		</div>

		<div class="border-t pt-6">
			<h2 class="text-lg font-semibold mb-4 text-gray-700">Upload SVG Template</h2>
			<input
				type="file"
				accept=".svg"
				on:change={handleFileUpload}
				class="block w-full text-sm text-gray-500 file:mr-4 file:py-2 file:px-4 file:rounded-md file:border-0 file:text-sm file:font-semibold file:bg-blue-50 file:text-blue-700 hover:file:bg-blue-100"
				disabled={!isValidColorCount}
			/>
			{#if !isValidColorCount}
				<p class="mt-2 text-sm text-amber-600">
					Please set the number of expected colors before uploading
				</p>
			{/if}
		</div>

		<div>
			<h2 class="text-lg font-semibold mb-4 text-gray-700">Color Palette</h2>
			<div class="flex flex-wrap gap-4 items-center">
				{#each colors as color, i}
					<div class="flex items-center gap-2 bg-gray-50 p-2 rounded-md">
						<input
							type="color"
							bind:value={colors[i]}
							class="w-14 h-10 rounded cursor-pointer border-2 border-gray-200"
						/>
						<button
							on:click={() => removeColor(i)}
							class="w-8 h-8 flex items-center justify-center text-gray-500 hover:text-red-500 hover:bg-gray-100 rounded-full transition-colors"
							title="Remove color"
						>
							×
						</button>
					</div>
				{/each}
			</div>
		</div>

		<div class="flex gap-3 items-center border-t pt-4">
			<input
				type="color"
				bind:value={newColor}
				class="w-14 h-10 rounded cursor-pointer border-2 border-gray-200"
			/>
			<button
				on:click={addColor}
				class="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 transition-colors duration-200 flex items-center gap-2 text-sm font-medium"
			>
				<span>Add New Color</span>
			</button>
		</div>
	</div>

	<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
		{#each combinations as combo}
			<div class="bg-white rounded-xl p-4 shadow-sm hover:shadow-md transition-shadow duration-200">
				<div class="aspect-square rounded-lg overflow-hidden bg-gray-50 flex items-center justify-center">
					{#if uploadedSvg === null}
						<p class="text-gray-400">Input SVG to start</p>
					{:else if combinations.length === 0}
						<p class="text-gray-400">Loading...</p>
					{:else}
						<img
							src={getSvgUrl(combo)}
							alt="Color combination pattern"
							class="w-full h-full object-cover"
						/>
					{/if}
				</div>
				<div class="flex gap-3 mt-4 px-2 flex-wrap">
					{#each combo as color}
						<div class="flex-shrink-0 text-center py-1 px-2 bg-gray-50 rounded text-xs font-mono text-gray-600 flex items-center gap-1">
							<div class="w-3 h-3 rounded-full" style="background-color: {color}" />
							{color}
						</div>
					{/each}
				</div>
			</div>
		{/each}
	</div>
</div>


<style>
	:global(object) :global(path:nth-child(1)) {
		fill: var(--color1) !important;
	}
	:global(object) :global(path:nth-child(2)) {
		fill: var(--color2) !important;
	}
	:global(object) :global(path:nth-child(3)) {
		fill: var(--color3) !important;
	}
</style>
