<script lang="ts">
	import {
		HillshadeLayer,
		MapLibre,
		RasterDEMTileSource,
		LineLayer,
		TerrainControl,
		Terrain,
		SymbolLayer
	} from 'svelte-maplibre-gl';
	import { MapLibreContourSource } from 'svelte-maplibre-gl/contour';
	import { decode, encode } from 'fast-png';

	function gsiToTerrainRGB(r: number, g: number, b: number): [number, number, number] {
		if (r == 128 && g == 0 && b == 0) return [1, 134, 160]; // NA値は0にマップ
		let x = 65536 * r + 256 * g + b;
		const elev = x < 8388608 ? x * 0.01 : (x - 16777216) * 0.01;
		let value = (elev + 10000) / 0.1;
		r = Math.floor(value / (256 * 256));
		value %= 256 * 256;
		g = Math.floor(value / 256);
		b = Math.floor(value % 256);
		return [r, g, b];
	}

	async function getTile(url: string, abortController: AbortController) {
		const options: RequestInit = { signal: abortController.signal };
		const response = await fetch(url, options);
		const output = new Uint8ClampedArray(256 * 256 * 4);
		if (!response.ok) {
			for (let i = 0; i < 256 * 256 * 4; i += 4) {
				[output[i], output[i + 1], output[i + 2], output[i + 3]] = [1, 134, 160, 255];
			}
		} else {
			const img = decode(await (await response.blob()).arrayBuffer());
			for (let i = 0; i < img.height * img.width; i++) {
				const o = i * 4;
				const p = i * 3;
				[output[o], output[o + 1], output[o + 2]] = gsiToTerrainRGB(
					img.data[p],
					img.data[p + 1],
					img.data[p + 2]
				);
				output[o + 3] = 255;
			}
		}
		return {
			data: new Blob([encode({ width: 256, height: 256, data: output })]),
			expires: response.headers.get('expires') || undefined,
			cacheControl: response.headers.get('cache-control') || undefined
		};
	}
</script>

<MapLibre
	class="relative h-screen w-screen"
	style="https://basemaps.cartocdn.com/gl/voyager-gl-style/style.json"
	zoom={10}
	pitch={45}
	minZoom={6}
	center={{ lng: 138.7, lat: 35.4 }}
>
	<div class="absolute top-3 left-3 z-10 rounded-md bg-white/80 p-3 leading-none backdrop-blur-sm">
		GitHub: <a class="underline" href="https://github.com/MIERUNE/maplibre-contour-gsi-demo"
			>MIERUNE/maplibre-contour-gsi-demo</a
		>
	</div>

	<MapLibreContourSource
		url={'https://cyberjapandata.gsi.go.jp/xyz/dem_png/{z}/{x}/{y}.png'}
		maxzoom={14}
		encoding={'mapbox'}
		tileOptions={{
			thresholds: {
				// zoom: [minor, major]
				9: [250, 500],
				11: [100, 500],
				12: [50, 200],
				13: [20, 100],
				14: [10, 50]
			},
			// optional, override vector tile parameters:
			contourLayer: 'contours',
			elevationKey: 'ele',
			levelKey: 'level'
		}}
		worker={false}
		{getTile}
		attribution="<a href='https://maps.gsi.go.jp/development/ichiran.html#dem'>国土地理院標高タイル</a>"
	>
		{#snippet children(demSource)}
			<RasterDEMTileSource
				id="dem"
				tiles={[demSource.sharedDemProtocolUrl]}
				maxzoom={14}
				tileSize={256}
				encoding="mapbox"
			>
				<TerrainControl exaggeration={1} />
				<Terrain source="dem" />
			</RasterDEMTileSource>
			<RasterDEMTileSource
				tiles={[demSource.sharedDemProtocolUrl]}
				maxzoom={14}
				tileSize={256}
				encoding="mapbox"
			>
				<HillshadeLayer
					paint={{
						'hillshade-exaggeration': 0.5,
						'hillshade-illumination-anchor': 'map',
						'hillshade-shadow-color': '#3080b0'
					}}
				/>
			</RasterDEMTileSource>
			<LineLayer
				sourceLayer="contours"
				paint={{
					'line-color': 'rgba(100, 40, 0, 1)',
					'line-opacity': 0.7,
					'line-width': ['match', ['get', 'level'], 1, 1.4, 0.8]
				}}
			/>
			<SymbolLayer
				sourceLayer="contours"
				filter={['>', ['get', 'level'], 0]}
				layout={{
					'symbol-placement': 'line',
					'text-size': 12,
					'text-field': ['number-format', ['get', 'ele'], {}]
				}}
				paint={{
					'text-halo-color': 'white',
					'text-halo-width': 1
				}}
			/>
		{/snippet}
	</MapLibreContourSource>
</MapLibre>

<svelte:head>
	<title>maplibre-contour + 地理院標高タイル</title>
</svelte:head>
