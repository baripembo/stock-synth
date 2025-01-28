<script>
	import { onMount, onDestroy } from 'svelte';
	import mapboxgl from 'mapbox-gl';
	import * as d3 from 'd3';
	import { legendColor } from 'd3-svg-legend';
	import * as turf from '@turf/turf';
	import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';

	mapboxgl.baseApiUrl = 'https://data.humdata.org/mapbox';
	mapboxgl.accessToken = 'cacheToken';

	export let center = [67, 33];
	export let zoom = 4.8;
	export let mapData = [];
	export let THEME;
  export let LOCATION;

	let map, mapContainer, geojson, mapLegend, colorScale, currentFeatures;
	let orgsObject = {};
	let sectorsObject = {};

  const colorRanges = {
    hno: ['#ffcdb2', '#f2b8aa', '#e4989c', '#b87e8b', '#925b7a']
  };
	
	let tooltip = d3.select('.tooltip');
	let numFormat = d3.format(',');
	let shortFormat = d3.format('.2s');
	let minZoom = 3;

	//map humanitarian icons to sector clusters
  let humIcons = {
    'Child Protection': 'humanitarianicons-Child-protection',
    'Camp Coordination / Management': 'humanitarianicons-Camp-Coordination-and-Camp-Management',
    'Coordination and Common Services': 'humanitarianicons-Coordination',
    'Early Recovery': 'humanitarianicons-Early-Recovery',
    'Education': 'humanitarianicons-Education',
    'Emergency Shelter and NFI': 'humanitarianicons-Shelter',
    'Emergency Telecommunications': 'humanitarianicons-Emergency-Telecommunications',
    'Food Security': 'humanitarianicons-Food-Security',
    'Gender Based Violence': 'humanitarianicons-Gender-based-violence',
    'Protection': 'humanitarianicons-Protection',
    'Health': 'humanitarianicons-Health',
    'Logistics': 'humanitarianicons-Logistics',
    'Mine Action': 'humanitarianicons-Mine',
    'Multi-Purpose Cash': 'humanitarianicons-Fund',
    'Nutrition': 'humanitarianicons-Nutrition',
    'Water Sanitation Hygiene': 'humanitarianicons-Water-Sanitation-and-Hygiene',
  };


	let data = mapData;
	$: if (data != mapData) {
		data = mapData;
    updateFeatures();
	}

	export const isMobile = () => {
    const userAgentCheck = /Mobi|Android/i.test(navigator.userAgent);
    const screenSizeCheck = window.matchMedia("(max-width: 767px)").matches;
    return userAgentCheck || screenSizeCheck;
	}

	onMount(() => {
		const mapHeight = isMobile() ? 400 : 600;
		mapContainer.style.height = `${mapHeight}px`;
    initializeMap();
	});

	function initializeMap() {
		//init map
	  map = new mapboxgl.Map({
	    container: mapContainer,
	    style: 'mapbox://styles/humdata/cl3lpk27k001k15msafr9714b',
      zoom,
      minZoom
	  });

	  map.addControl(new mapboxgl.NavigationControl({showCompass: false}))
	     .addControl(new mapboxgl.AttributionControl(), 'bottom-left');

	  tooltip = new mapboxgl.Popup({
			closeButton: false,
			closeOnClick: false,
			className: 'map-tooltip'
		});

	  map.on('load', function() {
	    console.log('Map loaded')
	  	loadFeatures();
	  });
	}

  function findAdmName(props, keys) {
    for (const key of keys) {
      if (props.hasOwnProperty(key)) {
        return props[key];
      }
    }
    return null;
  }

  function match_geojson(geojson, indicator_data) {
    geojson.features.forEach(feature => {
      const props = feature.properties;
      const matched_data = indicator_data.find(item => item.admin2_code === props.ADM2_PCODE);
      if (matched_data) {
        Object.assign(props, {
      		ADM1_NAME: matched_data.admin2_name,
        	indicator_value: matched_data.value,
        	indicator_name: THEME,
        	numOrgs: matched_data.numOrgs
        });
      }
      else {
      	props.ADM2_NAME = findAdmName(props, ['ADM2_EN', 'ADM2_ES', 'ADM2_FR', 'ADM2_PT']);
        props.indicator_value = 'NA';
        props.indicator_name = THEME;
      }
    });
    return geojson;
  }

	async function loadFeatures() {
    const colorRange = colorRanges[THEME] || colorRanges.population;

    //get geojson for current location
		const geojson_url = 'ssd.geojson';
		const response = await fetch(geojson_url);
	  const geojson_data = await response.json();
	  
	  currentFeatures = match_geojson(geojson_data, data);

  	const max = d3.max(currentFeatures.features, d => +d.properties.indicator_value);
  	colorScale = d3.scaleQuantize().domain([0, max]).range(colorRange);

	  currentFeatures.features.forEach(f => {
	    const prop = f.properties;
	    prop.color = prop.indicator_value==='NA' ? '#CCC' : colorScale(+prop.indicator_value)
	  });

	  map.addSource('indicator-data', {
	    type: 'geojson',
	    data: currentFeatures
	  });

	  map.addLayer({
	    id: 'indicator-layer',
	    source: 'indicator-data',
	    type: 'fill',
	    paint: {
      	'fill-color': ['get', 'color'],
	      'fill-outline-color': '#FFFFFF'
	    }
	  }, 'Countries 2-4');

	  //mouse events
	  map.on('mouseenter', 'indicator-layer', onMouseEnter);
	  map.on('mouseleave', 'indicator-layer', onMouseLeave);
	  map.on('mousemove', 'indicator-layer', onMouseMove);

	  zoomToBounds();
	  createMapLegend();
	}

	function updateFeatures() {
		//clear indicator layer on map
		if (map.getSource('indicator-data')) {
			map.removeLayer('indicator-layer');
			map.removeSource('indicator-data');
			loadFeatures();
		}

		//update legend
		d3.select('.legend-body').selectAll('*').remove();
		createMapLegend();
	}

	function zoomToBounds() {
		//zoom map to bounds
		const pad = isMobile() ? {top: 20, right: 20, bottom: 50, left: 20} : {top: 50, right: 50, bottom: 125, left: 50};
		if (currentFeatures) {
			const bbox = turf.bbox(currentFeatures);
			map.fitBounds(bbox, {padding: pad, duration: 100});
		}
	}


	function createMapLegend() {
		const legendTitle = getLegendTitle();
		d3.select('.legend-title').text(legendTitle);

	  const svg = d3.select(mapLegend);
		const colorLegend = legendColor()
	    .labelFormat(d3.format('.2s'))
	    .scale(colorScale);

		d3.select('.legend-body').call(colorLegend);

		//no data key
	  const nodata = svg.append('svg').attr('class', 'no-data-key');
	  nodata.append('rect').attr('width', 15).attr('height', 15);
	  nodata.append('text').attr('class', 'label').text('No Data');
	}


  function getLegendTitle() {
  	return 'Number of People in Need in WASH sector';
  }

	//mouse event/leave events
	function onMouseEnter(e) {
	  map.getCanvas().style.cursor = 'pointer';
	  tooltip.addTo(map);
	}

	function onMouseLeave(e) {
	  map.getCanvas().style.cursor = '';
	  tooltip.remove();
	}

  function onMouseMove(e) {
    const prop = e.features[0].properties;
    const adm0_name = findAdmName(prop, ['ADM0_EN', 'ADM0_ES', 'ADM0_FR', 'ADM0_PT']);
    let content = `<h2><strong>${prop.ADM1_NAME}</strong>, ${adm0_name}</h2>`;

    if (prop.indicator_value === 'NA') {
      content += `<span class='theme'>People in need:</span><div class="stat">No data</div>`;
    } else {
      content += `<span class='theme'>People in need in WASH sector:</span><div class="stat">${shortFormat(prop.indicator_value)}</div>`;
      content += `<hr><ul class="sector-list"><li><i class="${humIcons['Water Sanitation Hygiene']}"></i> WASH organizations present: ${prop.numOrgs}</li></ul>`;
    }

    tooltip.setHTML(content).addTo(map).setLngLat(e.lngLat);
  }

	onDestroy(() => {
		if (map) {
	  	map.remove();
		}
	});
</script>


<div class='right-content'>
	<div bind:this={mapContainer} />
	<div class='map-legend' bind:this={mapLegend}>
		<h4 class='legend-title'>Map Legend</h4>
		<svg>
			<g class='legend-body'></g>
		</svg>
	</div>
</div>

<style lang='scss'>
</style>