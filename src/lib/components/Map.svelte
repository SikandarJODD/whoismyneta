<script>
	import {
		geoMercator,
		scaleQuantile,
		scaleOrdinal,
		schemeOrRd,
		schemeBlues,
		schemeGreens,
		schemeSpectral,
		zoom,
		interpolateBlues,
		scaleSequential,
		scaleLinear,
		geoPath,
		scaleQuantize,
		scaleThreshold
	} from 'd3';
	import { feature } from 'topojson-client';
	import { Chart, Legend, Svg, GeoPath, Text, Transform } from 'layerchart';
	import country from '$lib/data/india_ls_seats_545.json';
	import { cubicOut } from 'svelte/easing';
	import TransformControls from './TransformControls.svelte';
	import { selectedConstituency } from '$lib/store';
	import { setConstituency } from '$lib/utils';
	import data from '$lib/data/data.json';
	import ToggleMap from './ToggleMap.svelte';

	let hovered = null;
	const states = feature(country, country.objects.india_ls_seats_545);

	let colorScale;
	let numericColorScale, categoricalColorScale;
	let transform;

	let category = {
		all: {
			key: 'all',
			type: 'all',
			label: 'All',
			color: 'bg-surface-100',
			text: 'black'
		},
		assets: {
			key: 'total_assets',
			label: 'Declared Assets',
			type: 'number',
			scheme: schemeOrRd[7],
			color: 'bg-orangePrimary',
			text: 'white',
			format: (d) => '₹' + parseInt(d / 10000000, 10) + ' cr'
		},
		criminalCases: {
			key: 'criminal_cases',
			type: 'number',
			label: 'Criminal Cases',
			scheme: schemeBlues[7],
			color: 'bg-bluePrimary',
			text: 'white',
			tickValues: [0, 1, 2, 3],
			format: (d) => parseInt(d, 10) + (d === 5 ? '+' : '')
		},
		attendance: {
			key: 'attendance',
			type: 'number',
			label: 'Attendance',
			scheme: schemeGreens[5],
			color: 'bg-sagePrimary',
			text: 'white',
			format: (d) => parseInt(d, 10) + '%'
		},
		education: {
			key: 'education_x',
			type: 'categorical',
			label: 'Education',
			color: 'bg-yellowPrimary',
			text: 'black'
		}
	};
	let selectedCategory = category.all;
	let assetScale;
	let criminalCasesScale;
	let attendanceScale;

	const educationLevels = [
		'Others',

		'10th Pass',
		'12th Pass',
		'Graduate',
		'Graduate Professional',
		'Post Graduate',
		'Doctorate'
	];

	$: if (selectedCategory.key === 'total_assets') {
		assetScale = scaleQuantile()
			.domain(data.map((d) => d[selectedCategory.key]))
			.range(selectedCategory.scheme);
	}

	$: if (selectedCategory.key === 'criminal_cases') {
		criminalCasesScale = scaleLinear().domain([0, 1, 2, 3, 4, 5]).range(selectedCategory.scheme);
	}

	$: if (selectedCategory.key === 'attendance') {
		attendanceScale = scaleQuantile()
			.domain(data.map((d) => d[selectedCategory.key]))
			.range(selectedCategory.scheme);
	}

	$: if (
		selectedCategory &&
		selectedCategory.key != 'all' &&
		selectedCategory.type === 'categorical'
	) {
		categoricalColorScale = scaleOrdinal(educationLevels, schemeSpectral[educationLevels.length]);
	}

	$: colorScale =
		selectedCategory.type === 'all'
			? undefined
			: selectedCategory.key === 'total_assets'
				? assetScale
				: selectedCategory.key === 'criminal_cases'
					? criminalCasesScale
					: selectedCategory.key === 'attendance'
						? attendanceScale
						: categoricalColorScale;

	// $: if ($selectedConstituency) {
	// 	const selected = states.features.find(
	// 		(feature) => feature.properties.ls_seat_name === $selectedConstituency.ls_seat_name
	// 	);
	// 	if (selected && transform) {
	// 		const geoMercatorProjection = geoMercator().fitSize([650, 900], selected);

	// 		const path = geoPath().projection(geoMercatorProjection);
	// 		const [[left, top], [right, bottom]] = path.bounds(selected);
	// 		const width = right - left;
	// 		const height = bottom - top;
	// 		const x = (left + right) / 2;
	// 		const y = (top + bottom) / 2;

	// 		// Calculate zoom level based on the bounding box dimensions
	// 		const padding = 20;
	// 		const zoomFactor = Math.min(400 / (width + padding), 800 / (height + padding));

	// 		// Adjust the center point based on the padding
	// 		const centerX = (left + right) / 2 - (400 / zoomFactor - width) / 2;
	// 		const centerY = (top + bottom) / 2 - (800 / zoomFactor - height) / 2;

	// 		transform.zoomTo({ x: x, y: y }, { width: width * 0.1, height: height * 0.1 });
	// 	}
	// }
</script>

<div class="overflow-auto">
	<ToggleMap on:change={(e) => (selectedCategory = category[e.detail.value])} options={category} />
</div>

<main
	class="border-[1px] border-surface-300 p-6 relative overflow-clip h-[360px] md:h-[650px] w-full max-w-[900px]"
>
	<div class="absolute top-1/2 md:top-2 right-2">
		<TransformControls {transform} />
	</div>

	<Chart
		geo={{
			projection: geoMercator,
			fitGeojson: states
		}}
	>
		<Svg>
			<Transform
				scroll="scale"
				bind:this={transform}
				tweened={{ duration: 800, easing: cubicOut }}
				let:zoomTo
				let:reset
				let:scale
			>
				<g class="states">
					{#each states.features as feature}
						<GeoPath
							on:click={(e) => {
								const { geoPath, event } = e.detail;

								let [[left, top], [right, bottom]] = geoPath.bounds(feature);
								selectedCategory = category.all;
								setConstituency(feature.properties.ls_seat_name);
								let width = right - left;
								let height = bottom - top;
								let x = (left + right) / 2;
								let y = (top + bottom) / 2;
								const padding = 80;
								console.log(left, top, right, bottom);
								zoomTo({ x, y }, { width: width + padding, height: height + padding });
							}}
							on:mousemove={() => (hovered = feature)}
							on:mouseleave={() => (hovered = null)}
							geojson={feature}
							stroke-width={0.4 / scale}
							stroke="#8F96A3"
							fill={colorScale &&
							// also check if total_assets is not undefined
							data.find((d) => d.ls_seat_name === feature.properties.ls_seat_name).total_assets !==
								undefined
								? colorScale(
										data.find((d) => d.ls_seat_name === feature.properties.ls_seat_name)[
											selectedCategory.key
										]
									)
								: '#ECEFF4'}
							style="fill-opacity:0.8"
							class="{$selectedConstituency &&
							$selectedConstituency.ls_seat_name === feature.properties.ls_seat_name
								? 'fill-neutral-900'
								: ''} hover:cursor-pointer 
							transition-colors duration-200 ease-in-out 
                           
							{data.find((d) => d.ls_seat_name === feature.properties.ls_seat_name).total_assets === undefined
								? 'pointer-events-none'
								: 'pointer-events-auto'}
                            hover:fill-neutral-50"
						/>
					{/each}
				</g>
				<g class="pointer-events-none labels">
					{#each states.features as feature}
						<GeoPath geojson={feature} let:geoPath>
							{@const [x, y] = geoPath.centroid(feature)}
							{#if $selectedConstituency && $selectedConstituency.ls_seat_name === feature.properties.ls_seat_name}
								<Text
									{x}
									scaleToFit
									y={y - 20}
									value={feature.properties.ls_seat_name}
									textAnchor="middle"
									verticalAnchor="start"
									class=" stroke-surface-100 font-serif  text-[5px] stroke-[2px] shadow-md"
								/>
							{/if}
						</GeoPath>
					{/each}
				</g>
			</Transform>
		</Svg>
		<div class="bg-white legend z-[1000]">
			{#if selectedCategory.type === 'categorical'}
				<Legend
					scale={colorScale}
					let:values
					let:scale
					classes={{
						root: 'bg-white  text-xs absolute py-2 px-4 rounded shadow-sm z-[9000]'
					}}
					tickValues={selectedCategory.tickValues}
					title={selectedCategory.label}
				>
					<div class="flex flex-col gap-1 bg-white">
						{#each values as value}
							{#if value !== undefined}
								{@const scaledValue = selectedCategory.valueScale
									? selectedCategory.valueScale[value]
									: value}
								<div class="flex items-center gap-1">
									<div class="rounded-full size-2" style:background-color={scale(scaledValue)} />
									<div class="text-xs text-surface-content/50">{value}</div>
								</div>
							{/if}
						{/each}
					</div>
				</Legend>
			{:else if selectedCategory.type === 'number'}
				{#key selectedCategory.key}
					<div class="legend">
						<Legend
							scale={colorScale}
							tickFormat={selectedCategory.format}
							classes={{
								root: 'bg-white  text-xs absolute py-2 px-4 rounded shadow-sm z-[9000]',
								label: 'anchor-end'
							}}
							title={selectedCategory.label}
						></Legend>
					</div>
				{/key}
			{/if}
		</div>
	</Chart>
</main>

<style>
	:global(.legend *) {
		text-anchor: end !important;
	}
</style>
