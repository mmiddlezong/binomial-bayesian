<script lang="ts">
	// Svelte 5 reactive state using runes
	type InputMode = 'wins-losses' | 'games-winrate';

	let inputMode = $state<InputMode>('wins-losses');
	let wins = $state(10);
	let losses = $state(5);
	let totalGames = $state(15);
	let winRate = $state(66.7);

	// Derived wins and losses based on input mode
	const actualWins = $derived(() => {
		if (inputMode === 'wins-losses') {
			return Math.max(0, wins);
		} else {
			const calculated = Math.round(
				(Math.max(0, Math.min(100, winRate)) / 100) * Math.max(0, totalGames)
			);
			return Math.max(0, calculated);
		}
	});

	const actualLosses = $derived(() => {
		if (inputMode === 'wins-losses') {
			return Math.max(0, losses);
		} else {
			const calculated =
				Math.max(0, totalGames) -
				Math.round((Math.max(0, Math.min(100, winRate)) / 100) * Math.max(0, totalGames));
			return Math.max(0, calculated);
		}
	});

	// Beta distribution PDF calculation
	function betaPDF(x: number, alpha: number, beta: number): number {
		// Using gamma function approximation for Beta distribution
		// PDF = x^(alpha-1) * (1-x)^(beta-1) / B(alpha, beta)
		// B(alpha, beta) = Gamma(alpha) * Gamma(beta) / Gamma(alpha + beta)

		// Handle boundary cases properly
		if (x < 0 || x > 1) return 0;

		// At x=0: only non-zero if alpha=1 (otherwise x^(alpha-1) -> 0)
		if (x === 0) {
			if (alpha === 1) {
				const logBeta = logGamma(alpha) + logGamma(beta) - logGamma(alpha + beta);
				return Math.exp(-logBeta); // (1-0)^(beta-1) / B(alpha, beta)
			}
			return 0;
		}

		// At x=1: only non-zero if beta=1 (otherwise (1-x)^(beta-1) -> 0)
		if (x === 1) {
			if (beta === 1) {
				const logBeta = logGamma(alpha) + logGamma(beta) - logGamma(alpha + beta);
				return Math.exp(-logBeta); // x^(alpha-1) / B(alpha, beta)
			}
			return 0;
		}

		const logBeta = logGamma(alpha) + logGamma(beta) - logGamma(alpha + beta);
		const logPDF = (alpha - 1) * Math.log(x) + (beta - 1) * Math.log(1 - x) - logBeta;

		return Math.exp(logPDF);
	}

	// Stirling's approximation for log-gamma function
	function logGamma(z: number): number {
		// Ensure z is positive to prevent NaN
		if (z <= 0 || !isFinite(z)) return NaN;

		if (z < 0.5) {
			// Use reflection formula for small z
			const sinTerm = Math.sin(Math.PI * z);
			if (sinTerm === 0) return NaN; // Avoid division by zero
			return Math.log(Math.PI) - Math.log(Math.abs(sinTerm)) - logGamma(1 - z);
		}

		// Stirling's approximation
		const g = 7;
		const coef = [
			0.99999999999980993, 676.5203681218851, -1259.1392167224028, 771.32342877765313,
			-176.61502916214059, 12.507343278686905, -0.13857109526572012, 9.9843695780195716e-6,
			1.5056327351493116e-7
		];

		z -= 1;
		let x = coef[0];
		for (let i = 1; i < g + 2; i++) {
			x += coef[i] / (z + i);
		}

		const t = z + g + 0.5;
		return Math.log(Math.sqrt(2 * Math.PI)) + Math.log(x) - t + (z + 0.5) * Math.log(t);
	}

	// Calculate posterior distribution data points
	function calculatePosterior() {
		// Using Beta(1,1) uniform prior
		// Posterior is Beta(alpha + wins, beta + losses)
		const alpha = Math.max(1, 1 + actualWins());
		const beta = Math.max(1, 1 + actualLosses());

		const points: Array<{ x: number; y: number }> = [];
		const numPoints = 200;

		for (let i = 0; i <= numPoints; i++) {
			const x = i / numPoints;
			const y = betaPDF(x, alpha, beta);
			// Filter out NaN or Infinity values
			if (isFinite(y)) {
				points.push({ x, y });
			}
		}

		return points;
	}

	// Calculate statistics
	function calculateStats() {
		const alpha = 1 + actualWins();
		const beta = 1 + actualLosses();
		const total = alpha + beta;

		// Mean of Beta distribution
		const mean = alpha / total;

		// Mode of Beta distribution
		let mode: number;
		if (alpha > 1 && beta > 1) {
			// Standard case: mode = (alpha - 1) / (total - 2)
			mode = (alpha - 1) / (total - 2);
		} else if (alpha <= 1 && beta > 1) {
			// Mode at x=0 when alpha <= 1 and beta > 1
			mode = 0;
		} else if (alpha > 1 && beta <= 1) {
			// Mode at x=1 when beta <= 1 and alpha > 1
			mode = 1;
		} else {
			// alpha <= 1 and beta <= 1: distribution is U-shaped or uniform
			// Use mean as a reasonable fallback
			mode = mean;
		}

		// Standard deviation
		const variance = (alpha * beta) / (total * total * (total + 1));
		const std = Math.sqrt(variance);

		// 95% credible interval (approximation using normal distribution for large samples)
		// For more accurate results, would need to use Beta quantile function
		const ci_lower = Math.max(0, mean - 1.96 * std);
		const ci_upper = Math.min(1, mean + 1.96 * std);

		return { mean, mode, std, ci_lower, ci_upper };
	}

	// Generate SVG path for the distribution
	function generatePath(
		points: Array<{ x: number; y: number }>,
		width: number,
		height: number
	): string {
		if (points.length === 0) return '';

		const maxY = Math.max(...points.map((p) => p.y));
		const padding = 40;
		const plotWidth = width - 2 * padding;
		const plotHeight = height - 2 * padding;

		const scaleX = (x: number) => padding + x * plotWidth;
		const scaleY = (y: number) => height - padding - (y / maxY) * plotHeight;

		let path = `M ${scaleX(points[0].x)} ${scaleY(points[0].y)}`;

		for (let i = 1; i < points.length; i++) {
			path += ` L ${scaleX(points[i].x)} ${scaleY(points[i].y)}`;
		}

		return path;
	}

	// Reactive derivations using $derived
	const posteriorPoints = $derived(calculatePosterior());
	const stats = $derived(calculateStats());
	const svgWidth = 700;
	const svgHeight = 400;
	const pathData = $derived(generatePath(posteriorPoints, svgWidth, svgHeight));

	// Grid calculations
	const padding = 40;
	const plotWidth = svgWidth - 2 * padding;
	const plotHeight = svgHeight - 2 * padding;
	const gridSpacing = 50;
	const numVerticalGridLines = Math.floor(plotWidth / gridSpacing) + 1;
	const numHorizontalGridLines = Math.floor(plotHeight / gridSpacing) + 1;
</script>

<div class="min-h-screen bg-neutral-950 p-8 font-sans">
	<div class="max-w-4xl mx-auto">
		<div class="text-center mb-12">
			<h1 class="text-4xl font-light text-neutral-100 mb-2 tracking-tight">Win Rate Estimator</h1>
			<p class="text-neutral-500">Bayesian inference from experimental data</p>
		</div>

		<!-- Input Controls -->
		<div class="bg-neutral-900 border border-neutral-800 rounded-xl p-6 mb-6">
			<!-- Input Mode Selector -->
			<div class="mb-8">
				<div class="text-xs uppercase tracking-wider font-semibold text-neutral-500 mb-3">
					Input Mode
				</div>
				<div class="flex gap-4">
					<button
						onclick={() => (inputMode = 'wins-losses')}
						class="flex-1 px-4 py-2.5 rounded-lg text-sm font-medium transition-all {inputMode ===
						'wins-losses'
							? 'bg-neutral-100 text-neutral-900'
							: 'bg-neutral-800 text-neutral-400 hover:bg-neutral-700 hover:text-neutral-200'}"
					>
						Wins & Losses
					</button>
					<button
						onclick={() => (inputMode = 'games-winrate')}
						class="flex-1 px-4 py-2.5 rounded-lg text-sm font-medium transition-all {inputMode ===
						'games-winrate'
							? 'bg-neutral-100 text-neutral-900'
							: 'bg-neutral-800 text-neutral-400 hover:bg-neutral-700 hover:text-neutral-200'}"
					>
						Games & Win Rate
					</button>
				</div>
			</div>

			<!-- Input Fields -->
			{#if inputMode === 'wins-losses'}
				<div class="grid grid-cols-2 gap-6">
					<div>
						<label
							for="wins"
							class="block text-xs uppercase tracking-wider font-semibold text-neutral-500 mb-2"
						>
							Wins
						</label>
						<input
							id="wins"
							type="number"
							bind:value={wins}
							oninput={(e) => {
								const val = parseFloat(e.currentTarget.value);
								if (!isNaN(val) && val < 0) wins = 0;
							}}
							min="0"
							class="w-full px-4 py-3 bg-neutral-800 border border-neutral-700 text-neutral-100 rounded-lg focus:ring-2 focus:ring-neutral-600 focus:border-transparent outline-none transition-all"
						/>
					</div>
					<div>
						<label
							for="losses"
							class="block text-xs uppercase tracking-wider font-semibold text-neutral-500 mb-2"
						>
							Losses
						</label>
						<input
							id="losses"
							type="number"
							bind:value={losses}
							oninput={(e) => {
								const val = parseFloat(e.currentTarget.value);
								if (!isNaN(val) && val < 0) losses = 0;
							}}
							min="0"
							class="w-full px-4 py-3 bg-neutral-800 border border-neutral-700 text-neutral-100 rounded-lg focus:ring-2 focus:ring-neutral-600 focus:border-transparent outline-none transition-all"
						/>
					</div>
				</div>
			{:else}
				<div class="grid grid-cols-2 gap-6">
					<div>
						<label
							for="totalGames"
							class="block text-xs uppercase tracking-wider font-semibold text-neutral-500 mb-2"
						>
							Total Games
						</label>
						<input
							id="totalGames"
							type="number"
							bind:value={totalGames}
							oninput={(e) => {
								const val = parseFloat(e.currentTarget.value);
								if (!isNaN(val) && val < 0) totalGames = 0;
							}}
							min="0"
							class="w-full px-4 py-3 bg-neutral-800 border border-neutral-700 text-neutral-100 rounded-lg focus:ring-2 focus:ring-neutral-600 focus:border-transparent outline-none transition-all"
						/>
					</div>
					<div>
						<label
							for="winRate"
							class="block text-xs uppercase tracking-wider font-semibold text-neutral-500 mb-2"
						>
							Win Rate (%)
						</label>
						<input
							id="winRate"
							type="number"
							bind:value={winRate}
							oninput={(e) => {
								const val = parseFloat(e.currentTarget.value);
								if (!isNaN(val)) {
									if (val < 0) winRate = 0;
									else if (val > 100) winRate = 100;
								}
							}}
							min="0"
							max="100"
							step="0.1"
							class="w-full px-4 py-3 bg-neutral-800 border border-neutral-700 text-neutral-100 rounded-lg focus:ring-2 focus:ring-neutral-600 focus:border-transparent outline-none transition-all"
						/>
					</div>
				</div>
			{/if}

			<!-- Statistics Display -->
			<div class="mt-8 grid grid-cols-2 md:grid-cols-5 gap-4">
				<div class="bg-neutral-800/50 border border-neutral-800 rounded-lg p-4">
					<div class="text-xs text-neutral-500 mb-1">Total Games</div>
					<div class="text-xl font-medium text-neutral-200">{actualWins() + actualLosses()}</div>
				</div>
				<div class="bg-neutral-800/50 border border-neutral-800 rounded-lg p-4">
					<div class="text-xs text-neutral-500 mb-1">Wins / Losses</div>
					<div class="text-xl font-medium text-neutral-200">{actualWins()} / {actualLosses()}</div>
				</div>
				<div class="bg-neutral-800/50 border border-neutral-800 rounded-lg p-4">
					<div class="text-xs text-neutral-500 mb-1">Mean Win Rate</div>
					<div class="text-xl font-medium text-neutral-200">
						{(stats.mean * 100).toFixed(1)}%
					</div>
				</div>
				<div class="bg-neutral-800/50 border border-neutral-800 rounded-lg p-4">
					<div class="text-xs text-neutral-500 mb-1">Mode</div>
					<div class="text-xl font-medium text-neutral-200">
						{(stats.mode * 100).toFixed(1)}%
					</div>
				</div>
				<div class="bg-neutral-800/50 border border-neutral-800 rounded-lg p-4">
					<div class="text-xs text-neutral-500 mb-1">Std Dev</div>
					<div class="text-xl font-medium text-neutral-200">
						{(stats.std * 100).toFixed(1)}%
					</div>
				</div>
			</div>

			<!-- Credible Interval -->
			<div
				class="mt-4 bg-neutral-800/30 border border-neutral-800 rounded-lg p-4 flex items-center justify-between"
			>
				<div>
					<div class="text-xs text-neutral-500 mb-1">95% Credible Interval</div>
					<div class="text-lg font-medium text-neutral-300">
						{(stats.ci_lower * 100).toFixed(1)}% – {(stats.ci_upper * 100).toFixed(1)}%
					</div>
				</div>
				<div class="text-xs text-neutral-600 max-w-[200px] text-right hidden sm:block">
					95% probability the true win rate lies within this range
				</div>
			</div>
		</div>

		<!-- Distribution Plot -->
		<div class="bg-neutral-900 border border-neutral-800 rounded-xl p-6">
			<h2 class="text-lg font-medium text-neutral-200 mb-6">Posterior Distribution</h2>

			<div class="w-full flex justify-center py-4">
				<svg
					width={svgWidth}
					height={svgHeight}
					viewBox="0 0 {svgWidth} {svgHeight}"
					preserveAspectRatio="xMidYMid meet"
					class="max-w-full h-auto"
				>
					<!-- Background -->
					<rect width={svgWidth} height={svgHeight} fill="#171717" />

					<!-- Grid lines -->
					{#each Array(numVerticalGridLines) as _, i}
						<line
							x1={padding + i * gridSpacing}
							y1={padding}
							x2={padding + i * gridSpacing}
							y2={svgHeight - padding}
							stroke="#262626"
							stroke-width="1"
						/>
					{/each}
					{#each Array(numHorizontalGridLines) as _, i}
						<line
							x1={padding}
							y1={padding + i * gridSpacing}
							x2={svgWidth - padding}
							y2={padding + i * gridSpacing}
							stroke="#262626"
							stroke-width="1"
						/>
					{/each}

					<!-- Axes -->
					<line
						x1={padding}
						y1={svgHeight - padding}
						x2={svgWidth - padding}
						y2={svgHeight - padding}
						stroke="#525252"
						stroke-width="1"
					/>
					<line
						x1={padding}
						y1={padding}
						x2={padding}
						y2={svgHeight - padding}
						stroke="#525252"
						stroke-width="1"
					/>

					<!-- X-axis labels -->
					{#each [0, 0.25, 0.5, 0.75, 1.0] as tick}
						<text
							x={padding + tick * plotWidth}
							y={svgHeight - 20}
							text-anchor="middle"
							class="text-xs fill-neutral-500"
						>
							{(tick * 100).toFixed(0)}%
						</text>
					{/each}

					<!-- Axis labels -->
					<text
						x={svgWidth / 2}
						y={svgHeight - 5}
						text-anchor="middle"
						class="text-xs uppercase tracking-widest fill-neutral-600 font-medium"
					>
						Win Rate
					</text>

					<text
						x="15"
						y={svgHeight / 2}
						text-anchor="middle"
						transform="rotate(-90, 15, {svgHeight / 2})"
						class="text-xs uppercase tracking-widest fill-neutral-600 font-medium"
					>
						Probability Density
					</text>

					<!-- Distribution curve -->
					<path
						d={pathData}
						stroke="#e5e5e5"
						stroke-width="2"
						fill="none"
						stroke-linecap="round"
						stroke-linejoin="round"
					/>

					<!-- Fill under curve -->
					{#if pathData}
						<path
							d={pathData +
								` L ${svgWidth - padding} ${svgHeight - padding} L ${padding} ${svgHeight - padding} Z`}
							fill="#e5e5e5"
							fill-opacity="0.05"
						/>
					{/if}

					<!-- Mean line -->
					<line
						x1={padding + stats.mean * plotWidth}
						y1={padding}
						x2={padding + stats.mean * plotWidth}
						y2={svgHeight - padding}
						stroke="#737373"
						stroke-width="1"
						stroke-dasharray="4,4"
					/>
				</svg>
			</div>

			<div class="mt-6 text-sm text-neutral-500 flex justify-between items-center">
				<p>
					Posterior distribution (Beta({1 + actualWins()}, {1 + actualLosses()}))
				</p>
				<div class="flex items-center gap-2">
					<div class="w-3 h-px bg-neutral-400 border-t border-dashed border-neutral-400"></div>
					<span class="text-xs">Mean estimate</span>
				</div>
			</div>
		</div>

		<!-- Explanation -->
		<div class="mt-6 p-6 border border-neutral-800 rounded-xl">
			<h3 class="text-sm font-semibold text-neutral-300 uppercase tracking-wider mb-4">
				How it works
			</h3>
			<ul class="space-y-3 text-sm text-neutral-500">
				<li class="flex gap-3">
					<span class="w-1.5 h-1.5 rounded-full bg-neutral-700 mt-1.5 shrink-0"></span>
					<span>
						<strong class="text-neutral-400">Prior:</strong> Uniform Beta(1,1) distribution (uninformed
						prior).
					</span>
				</li>
				<li class="flex gap-3">
					<span class="w-1.5 h-1.5 rounded-full bg-neutral-700 mt-1.5 shrink-0"></span>
					<span>
						<strong class="text-neutral-400">Likelihood:</strong> Binomial distribution from observed
						wins/losses.
					</span>
				</li>
				<li class="flex gap-3">
					<span class="w-1.5 h-1.5 rounded-full bg-neutral-700 mt-1.5 shrink-0"></span>
					<span>
						<strong class="text-neutral-400">Posterior:</strong> Beta({1 + actualWins()}, {1 +
							actualLosses()}) distribution combining prior and data.
					</span>
				</li>
			</ul>
		</div>
	</div>
</div>

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		background-color: #0a0a0a;
	}
</style>
