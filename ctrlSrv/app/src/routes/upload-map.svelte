<script lang="ts">
	import { goto } from '$app/navigation';
	import TextField from 'smelte/src/components/TextField';

	import { NEVER, Subject, switchMap, takeUntil, tap } from 'rxjs';
	import type { Observable } from 'rxjs';

	import Map from '../components/Map/Map.svelte';
	import Button from 'smelte/src/components/Button';
	import { mapBackgroundImage, mapMaxPosition } from '../store/map-background-image.store';
	import { onDestroy, onMount } from 'svelte';
	import { FileUploader, getBase64 } from '$src/utils/file-uploader.model';
	import Marker from '../components/Map/Marker.svelte';
	import { BEACON_ICON_URL } from '$src/streams/beacons/constants';
	import type { IIndoorPosition } from '$src/interfaces/position.interface';
	import type { IndoorMapEvents } from '$src/interfaces/indoor-map.interface';

	let fileUploader: FileUploader;

	const MAP_ROUTE = '/';

	const openUploadWindow = () => {
		fileUploader.openUploadWindow();
	};

	const unsubscribe = new Subject<void>();
	let imageUrl: Observable<string> = NEVER;
	let currentImageUrl: string = mapBackgroundImage.value;
	let mapSize: IIndoorPosition;

	let xDimension: string | null = mapMaxPosition.value?.x.toString() ?? null;
	let yDimension: string | null = mapMaxPosition.value?.y.toString() ?? null;

	let point1X = '1';
	let point1Y = '1';
	let point2X = '3';
	let point2Y = '5';

	$: {
		if (xDimension || yDimension) {
			mapSize = {
				x: +(xDimension as string),
				y: +(yDimension as string)
			};
		}
	}

	const listenForUploadedImages = (file: Observable<File>) => {
		imageUrl = file.pipe(
			takeUntil(unsubscribe),
			switchMap(getBase64),
			tap((image) => {
				currentImageUrl = image;
			})
		);
	};

	const updateInternalPositions = ({ x, y }: IIndoorPosition) => {
		xDimension = x.toString();
		yDimension = y.toString();
	};

	const handleMapUpdate = (event: CustomEvent<IndoorMapEvents['boundsUpdate']>) => {
		const position = event.detail;
		updateInternalPositions(position);
	};

	const saveMap = () => {
		if (xDimension && yDimension) {
			mapBackgroundImage.next(currentImageUrl);
			mapMaxPosition.next({
				x: +xDimension,
				y: +yDimension
			});
			goto(MAP_ROUTE);
		}
	};

	onMount(() => {
		fileUploader = new FileUploader();
		listenForUploadedImages(fileUploader.fileUpload);
	});

	onDestroy(() => {
		unsubscribe.next();
	});
</script>

<div class="container vertical-container">
	<section class="vertical-container">
		<p class="hint">Upload a Map if you wish to use your own.</p>
		<Button variant="raised" on:click={openUploadWindow}>Upload a Map</Button>
	</section>
	<!-- svelte-ignore a11y-missing-attribute -->
	<div class="map-wrapper">
		<Map backgroundImage={$imageUrl} {mapSize} on:boundsUpdate={handleMapUpdate}>
			<Marker x={+point1X} y={+point1Y} id="1" icon={BEACON_ICON_URL} />
			<Marker x={+point2X} y={+point2Y} id="2" icon={BEACON_ICON_URL} />
		</Map>
	</div>
	<section class="controls-group vertical-container">
		<p>Set the map dimensions in meters.</p>
		<div class="controls">
			<TextField label="X Dimension (Meters)" bind:value={xDimension} />
			<TextField label="Y Dimension (Meters)" bind:value={yDimension} />
		</div>
		<p>You can test the dimensions set by changing the coordinates of these two points.</p>
		<div class="controls point-coordinates">
			<TextField label="Point 1 X Coordinate (Meters)" bind:value={point1X} />
			<TextField label="Point 1 Y Coordinate (Meters)" bind:value={point1Y} />
		</div>
		<div class="controls point-coordinates">
			<TextField label="Point 2 X Coordinate (Meters)" bind:value={point2X} />
			<TextField label="Point 2 Y Coordinate (Meters)" bind:value={point2Y} />
		</div>
	</section>
	<Button variant="raised" on:click={saveMap} disabled={!(xDimension && yDimension)}>Save</Button>
	<style>
		.controls > * {
			margin: 0.5em;
		}
	</style>
</div>

<style>
	.vertical-container {
		display: flex;
		flex-direction: column;
		align-items: center;
	}
	.map-wrapper {
		width: 100%;
		border-radius: 4px;
		overflow: hidden;
	}
	.controls {
		display: flex;
	}
	section {
		margin: 2em 0;
	}

	.hint {
		margin-bottom: 1em;
	}
</style>
