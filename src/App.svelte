<script>
  import { onMount } from 'svelte';
  import axios from 'axios';
  import Papa from 'papaparse';
  import * as d3 from 'd3';

  import HNOView from './lib/HNOView.svelte';
  
  let viewsData = [];
  let detailsLoading = true;
  let abortController;

  let selectedTab = 1;

  let selectedCountry = 'SSD';

  const base_url = 'https://hapi.humdata.org/api/v1/';
  const app_indentifier = 'aGFwaS1kYXNoYm9hcmQ6ZXJpa2Eud2VpQHVuLm9yZw==';
  const rateDelay = 0;

  const countries = [
    { code: 'SSD', name: 'South Sudan' }
  ];

  const views = [
    {name: 'Operational Presence', id: 'orgs', endpoint: `coordination-context/operational-presence?sector_code=WSH&admin_level=2`},
    {name: 'Humanitarian Needs', id: 'hno', endpoint: `affected-people/humanitarian-needs?sector_code=WSH&population_status=INN&admin_level=2`},
  ];

  async function fetchData(endpoint) {
    try {
      const response = await fetch(`${endpoint}&app_identifier=${app_indentifier}`, { signal: abortController.signal });
      const text = await response.text();
      return new Promise((resolve, reject) => {
        Papa.parse(text, {
          header: true,
          complete: results => resolve(results.data),
          error: err => reject(err)
        });
      });
    } catch (error) {
      if (error.name === 'AbortError') {
        console.log('Fetch aborted');
      } else {
        throw error;
      }
    }
  }

  async function fetchDataWithRateLimit(endpoint, delay) {
    await new Promise(resolve => setTimeout(resolve, delay));
    return fetchData(endpoint);
  }

  function generateMetadataEndpoint(hdx_id) {
    return `${base_url}metadata/resource?resource_hdx_id=${hdx_id}&output_format=csv&limit=100`;
  }

  async function loadViewsData() {
    viewsData = [];
    let delay = 0;
    for (let view of views) {
      let data = [];
      let offset = 0;
      let fetchedData;
      let endpoint = '';

      //data pagination
      do {
        endpoint = `${base_url}${view.endpoint}&location_code=${selectedCountry}&offset=${offset}&output_format=csv&limit=10000`;
        fetchedData = await fetchDataWithRateLimit(endpoint, delay);
        data = data.concat(fetchedData);
        offset += 10000;
      } while (fetchedData.length >= 10000);

      //if no data
      if (data.length === 0) {
        viewsData = [...viewsData, { id: view.id, data: null, metadata: null }];
        continue;
      }

      //get metadata
      const metadataEndpoint = generateMetadataEndpoint(data[0].resource_hdx_id);
      const metadata = await fetchDataWithRateLimit(metadataEndpoint, delay + 100);

      //console.log('---view endpoint', endpoint)
      viewsData = [...viewsData, { id: view.id, data, metadata, endpoint }];

      delay += rateDelay;
    }
  }


  function selectTab(index) {
    selectedTab = index;
  }

  async function loadData() {
    await loadViewsData();
    detailsLoading = false;
  }  


  function initTracking() {
    //initialize mixpanel
    var MIXPANEL_TOKEN = window.location.hostname=='ocha-dap.github.io'? '5cbf12bc9984628fb2c55a49daf32e74' : '99035923ee0a67880e6c05ab92b6cbc0';
    mixpanel.init(MIXPANEL_TOKEN);
    mixpanel.track('page view', {
      'page title': document.title,
      'page type': 'datavis'
    });
  }

  onMount(async () => {
    abortController = new AbortController();
    loadData();
    initTracking();
  });
</script>


<main>
  <header>
    <a href='https://hdx-hapi.readthedocs.io/en/latest/' target='_blank'><img src='logo_hdx_hapi.png' alt='HDX HAPI' /></a>
  </header>
  
  <h2 class='header'><strong>South Sudan:</strong> People in Need in Water Sanitation Hygiene Sector</h2>

  <div class='tab-content'>
    {#if detailsLoading}
      <p class='no-data-msg'>Loading...</p>
    {:else}
      <HNOView {...viewsData[1]} orgData={viewsData[0].data} iso3={selectedCountry} />
    {/if}
  </div>
</main>


<style lang='scss'>
  header {
    display: flex;
    flex-flow: row;
    margin-top: 20px;
  }
  header img {
    height: 30px;
    margin-right: 30px;
  }
  header p {
    margin-top: 0;
    width: 60%;
  }
  h2.details {
    margin-top: 5px;
  }
  .main-content  {
    margin: 0;
  }
  select {
    font-size: 20px;
  }
  .tab-content {
    min-height: 621px;
  }

  @media (max-width: 768px) {
    header {
      flex-flow: column;
      img {
        height: 20px;
      }
      p {
        width: 100%;
      }
    }
    .select-wrapper {
      margin-left: 20px;
    }
  }
</style>
