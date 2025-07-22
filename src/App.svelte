<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let svg;
  let chartContainer;
  const races = ['Black', 'White', 'Latinx', 'Asian'];

  // We'll use this for the D3 chart data
  let assetData = [];

  // Utility: Transform "Hispanic" => "Latinx" for the chart race list
  function normalizeRace(race) {
    return race === "Hispanic" ? "Latinx" : race;
  }

  // Transform/filter on mount
  onMount(async () => {
    // *IMPORTANT*: Use a relative path if your app has a subdirectory base
    let raw = await d3.json('data/median_assets_by_race.json');
    if (raw && Array.isArray(raw)) {
      // Filter only "Total Assets" and our target races
      let filtered = raw.filter(
        d => d.metric_type === "Asset" &&
             d.metric === "Total Assets" &&
             (["Black", "White", "Hispanic", "Asian"].includes(d.racecl4))
      );

      // D3 expects "year": ..., "Black": ..., "White": ..., etc.
      // Transform long data to wide, renaming Hispanic→Latinx as needed
      let yearSet = Array.from(new Set(filtered.map(d => d.year))).sort();
      assetData = yearSet.map(year => {
        let entry = { year };
        filtered.forEach(d => {
          if (d.year === year) {
            let k = normalizeRace(d.racecl4);
            if (races.includes(k)) entry[k] = d.median;
          }
        });
        // Make sure all keys exist (for missing data)
        races.forEach(race => {
          if (!(race in entry)) entry[race] = null;
        });
        return entry;
      });

      initChart();
      observeSteps();
    }
  });

  function initChart() {
    const width = 500;
    const height = 400;

    // Remove prior SVG if remounting
    d3.select(chartContainer).selectAll("svg").remove();

    svg = d3.select(chartContainer)
      .append('svg')
      .attr('width', width)
      .attr('height', height);

    drawChart();
  }

  function drawChart(highlightRace = null) {
    if (!assetData.length) return;

    svg.selectAll('*').remove();

    // Define scales using the "wide" data
    const x = d3.scaleLinear()
      .domain(d3.extent(assetData, d => d.year))
      .range([60, 460]);

    const y = d3.scaleLinear()
      .domain([0, d3.max(assetData, d => d3.max(races.map(r => d[r])))])
      .range([360, 40]);

    // Draw a line for each race
    races.forEach(race => {
      svg.append('path')
        .datum(assetData.filter(d => d[race] !== null))
        .attr('fill', 'none')
        .attr('stroke', race === highlightRace ? '#e63946' : '#ccc')
        .attr('stroke-width', race === highlightRace ? 3 : 1.5)
        .attr('d', d3.line()
          .defined(d => d[race] !== null)
          .x(d => x(d.year))
          .y(d => y(d[race]))
        );
    });

    svg.append('g')
      .attr('transform', 'translate(0,360)')
      .call(d3.axisBottom(x).ticks(6).tickFormat(d3.format('d')));

    svg.append('g')
      .attr('transform', 'translate(60,0)')
      .call(d3.axisLeft(y));
  }

  function observeSteps() {
    const steps = document.querySelectorAll('.step');
    const observer = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          const step = (entry.target instanceof HTMLElement) ? entry.target.dataset.step : null;
          if (step === 'intro') drawChart();
          else if (step === 'national') drawChart('Black');
          else if (step === 'insights') drawChart('White');
          else if (step === 'policy') drawChart('Latinx');
        }
      });
    }, { threshold: 0.5 });

    steps.forEach(step => observer.observe(step));
  }
</script>

<style>
  .container { display: flex; flex-direction: column; align-items: center; max-width: 800px; margin: auto; padding: 2rem; }
  .graphic { position: sticky; top: 10px; height: 320px; margin-bottom: 50px; background: #f0f0f0; }
  .step { margin: 80px 0; padding: 2rem; background: #fff; border-radius: 10px; box-shadow: 0 0 8px rgba(0,0,0,0.1); }
  h2 { margin-bottom: 0.5rem; }
</style>

<div class="container">
  <div class="graphic" bind:this={chartContainer}></div>

  <div class="step" data-step="intro">
    <h2>Introduction</h2>
    <p>This story explores racial wealth disparities in the U.S. over time.</p>
  </div>
  <div class="step" data-step="national">
    <h2>Black Households</h2>
    <p>Tracking Black household median wealth from 2007 to 2019.</p>
  </div>
  <div class="step" data-step="insights">
    <h2>White Households</h2>
    <p>Comparative wealth trajectory for White households.</p>
  </div>
  <div class="step" data-step="policy">
    <h2>Latinx Households</h2>
    <p>Looking at the growth pattern of Latinx household wealth.</p>
  </div>
</div>
