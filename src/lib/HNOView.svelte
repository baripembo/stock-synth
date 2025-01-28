<script>
	import * as d3 from 'd3';
	import { onMount } from 'svelte';
  import Map from './Map.svelte'
  import Source from './Source.svelte'

	export let id;
	export let data;
	export let metadata;
  export let endpoint;
  export let iso3;
  export let orgData;

  let mapData = [];

	function formatData(data) {
    const pinAdm2Object = {};

    //format hno data
    data.forEach(item => {
      let { admin2_code, admin2_name, population_status, category, sector_name, sector_code, population } = item;

      if (!admin2_code) return;

      population = +population;

      //initialize admin2_code group if doesnt already exist
      if (!pinAdm2Object[admin2_code]) {
        pinAdm2Object[admin2_code] = {
          admin2_name,
          populationInnAll: 0
        };
      }

      //filter data
      if (sector_code === 'WSH' && category === 'total') {
        if (population_status === 'INN') {
          pinAdm2Object[admin2_code].populationInnAll += population;
        } 
      }
    });

    //format orgs data
    const admOrgs = {};
    const allOrgs = new Set();
    orgData.forEach(row => {
      const { admin2_code, admin2_name, org_name, sector_name } = row;

      if (!admOrgs[admin2_code]) {
        admOrgs[admin2_code] = {
          admin2_name,
          org_names: new Set(),
          sector_names: new Set(),
        };
      }

      if (org_name) {
        admOrgs[admin2_code].org_names.add(org_name);
        allOrgs.add(org_name);
      }
      if (sector_name) {
        admOrgs[admin2_code].sector_names.add(sector_name);
      }
    });

    //convert to array of objects
    const pinAdm2 = {};
    for (const [admin2_code, attr] of Object.entries(pinAdm2Object)) {
      let numOrgValue = (admOrgs[admin2_code]!==undefined) ? admOrgs[admin2_code].org_names.size : 0;
      pinAdm2[admin2_code] = {
        admin2_code: admin2_code,
        admin2_name: attr.admin2_name,
        value: attr.populationInnAll,
        numOrgs: numOrgValue
      };
    }

    mapData = Object.values(pinAdm2);
  }

 	onMount(() => {
    if (data) {
      formatData(data);
    }
	})
</script>


<div class='content grid-container'>

  {#if data && metadata}
    
    <div class='main-content col-12'>
      {#if mapData.length>1}
        <Map mapData={mapData} THEME={id} LOCATION={iso3} />
      {/if}
      <Source metadata={metadata[0]} align={'right'} />
    </div>

  {:else}

    <div class='col-12'>
      <p class='no-data-msg'>No data available for this view.</p>
    </div>

  {/if}

</div>

<style lang='scss'>
  .select-wrapper {
    width: 100px;
    select {
      height: 38px;
    }
  }
</style>
