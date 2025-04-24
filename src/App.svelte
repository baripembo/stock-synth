<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import * as Tone from 'tone/build/esm/index.js';


  // Props
  export let symbol = 'AAPL';
  export let outputSize = 'compact';
  export let startDate = null;    // e.g. '2024-01-01'
  export let endDate   = null;    // e.g. '2024-06-30'
  export let playSound = false;
  export let audioDuration = 30;  // total playback duration in seconds

  const apiKey       = 'IEW89S0LFSHPCFXN';
  const functionType = 'TIME_SERIES_DAILY';
  const seriesKey    = 'Time Series (Daily)';

  let data = [];
  let svg;
  let timeScale;
  let chart;
  let xScale;

  // Load data, draw chart, then optionally play sound
  async function loadAndDraw() {
    // Fetch

    const url = 'aapl.json';//`https://www.alphavantage.co/query?function=${functionType}&symbol=${symbol}&outputsize=compact&apikey=${apiKey}`;
    const json = await fetch(url).then(r => r.json());
    const timeSeries = json[seriesKey];
    if (!timeSeries) {
      console.error('API error or rate limit:', json);
      return;
    }

    // Transform
    data = Object.entries(timeSeries).map(([date, vals]) => ({
      date:  new Date(date),
      close: +vals['4. close']
    })).sort((a, b) => a.date - b.date);

    // Filter by date range
    if (startDate) {
      const start = new Date(startDate);
      data = data.filter(d => d.date >= start);
    }
    if (endDate) {
      const end = new Date(endDate);
      data = data.filter(d => d.date <= end);
    }

    drawChart();

    // Play sound mapping if enabled
    if (playSound && data.length) {
      console.log('am i here')
      await playDataSound();
    }
  }

  function drawChart() {
    const margin = { top: 20, right: 30, bottom: 30, left: 40 };
    const width  = 800 - margin.left - margin.right;
    const height = 400 - margin.top  - margin.bottom;

    // Clear SVG
    d3.select(svg).selectAll('*').remove();

    xScale = d3.scaleTime()
      .domain(d3.extent(data, d => d.date))
      .range([0, width]);

    const y = d3.scaleLinear()
      .domain([d3.min(data, d => d.close), d3.max(data, d => d.close)])
      .nice()
      .range([height, 0]);

    chart = d3.select(svg)
      .attr('width', width + margin.left + margin.right)
      .attr('height', height + margin.top + margin.bottom)
      .append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`);

    chart.append('g')
      .attr('transform', `translate(0,${height})`)
      .call(d3.axisBottom(xScale));

    chart.append('g')
      .call(d3.axisLeft(y));

    chart.append('path')
      .datum(data)
      .attr('fill', 'none')
      .attr('stroke', 'steelblue')
      .attr('stroke-width', 1.5)
      .attr('d', d3.line()
        .x(d => xScale(d.date))
        .y(d => y(d.close))
      );

    // Add play-head line at initial position
    chart.append('line')
      .attr('class', 'play-head')
      .attr('x1', 0)
      .attr('x2', 0)
      .attr('y1', 0)
      .attr('y2', height)
      .attr('stroke', 'red')
      .attr('stroke-width', 1);

    // Prepare timeScale for animation
    timeScale = d3.scaleTime()
      .domain(d3.extent(data, d => d.date))
      .range([0, audioDuration * 1000]); // ms
  }

  function animatePlayHead() {
    const start = Date.now();
    const end   = start + audioDuration * 1000;

    function tick() {
      const now = Date.now();
      const elapsed = now - start;
      if (elapsed > audioDuration * 1000) return;
      // Find corresponding date at elapsed
      const currentDate = timeScale.invert(elapsed);
      const xPos = xScale(currentDate);
      chart.select('.play-head')
        .attr('x1', xPos)
        .attr('x2', xPos);
      requestAnimationFrame(tick);
    }
    requestAnimationFrame(tick);
  }

  async function playDataSound() {
    console.log('playDataSound')
    await Tone.start();

    // Create a continuous oscillator
    const osc = new Tone.Oscillator({ type: 'sine', frequency: 0 }).toDestination();
    const now = Tone.now();

    // Scales for mapping
    const freqScale = d3.scaleLinear()
      .domain(d3.extent(data, d => d.close))
      .range([200, 800]);

    const timeScale = d3.scaleTime()
      .domain(d3.extent(data, d => d.date))
      .range([0, audioDuration]);

    // Initialize frequency at first data point
    osc.frequency.value = freqScale(data[0].close);
    osc.start(now);

    // Schedule frequency ramps along timeline
    data.forEach(d => {
      const t = now + timeScale(d.date);
      const f = freqScale(d.close);
      osc.frequency.linearRampToValueAtTime(f, t);
    });

    // Stop oscillator at end
    osc.stop(now + audioDuration);
  }

  function onBtnClick() {
    console.log('btn clicked')
    playSound = true;
    playDataSound();
    animatePlayHead();
  }

  // Initial load and redraw on prop change
  onMount(loadAndDraw);
  $: if (svg) loadAndDraw();
</script>

<div style="margin-bottom: 10px;">
  <button on:click={onBtnClick}>Play Timeline Sound</button>
</div>
<svg bind:this={svg}></svg>

<style>
  svg { width: 100%; height: auto; }
  button { padding: 6px 12px; }
</style>
