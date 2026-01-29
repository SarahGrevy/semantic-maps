<script>
  import { onMount } from 'svelte';
  import { scaleOrdinal } from 'd3-scale';
  import { schemeCategory10 } from 'd3-scale-chromatic';

  export let hoveredData;
  export let domainColumn;
  export let data;
  export let getStatusColor;
  
  let filteredData = [];
  let dates = [];
  let selectedDate = null;
  let isPlaying = false;
  let interval


  // const colorScale = scaleOrdinal(schemeCategory10);

  // onMount(() => {
  //     getStatusColor.domain(data.map(d => d[domainColumn]));
  // });
  
</script>

<div class="detail-card">
  {#if hoveredData}
  {#if hoveredData.title}
  <h1>{hoveredData.title}</h1>
  {/if}
  {#if hoveredData[domainColumn]}
  <span style="background: {getStatusColor(hoveredData[domainColumn])};">
      {hoveredData[domainColumn]}</span>
  {/if}
  {#if hoveredData.date}
  <h2>{hoveredData.date.toISOString().split('T')[0]}</h2>
  {/if}
  {#if hoveredData.text}
  <p>{hoveredData.text}</p>
  {/if}
  {:else}
  <p>Hover over a circle to see details here.</p>
  {/if}
</div>

<style>
  .detail-card {
    flex: 1;
    padding: 15px;
    margin-right: 30px;
    margin-top: 20px;
    border-top: 1px solid rgba(0, 0, 0, 1);
    /* box-shadow: 0 2px 5px rgba(0,0,0,0.3); */
    height: fit-content;
    height: 500px;
    width: 800px;
    overflow-y: auto;
    line-height: 1.5;
  }


  h1,h2 {
      margin: 0;
      padding: 0;
      font-weight: 300;
      margin-bottom: 10px;
      }

  h1 {
      font-size: 1.25rem;
      font-weight: 400;
      margin-bottom: 6px;
      width: 80%;
      margin-bottom: 10px;
      }

  h2 {
      font-size: 1rem;
      text-transform: uppercase;
      }

  span {
      padding: 6px 6px;
      display: inline-block;
      vertical-align: bottom;
      border-radius: 3px;
      color: rgba(0, 0, 0, 0.7);
      color: white;
      margin-bottom: 10px;
      }

  p {
          font-size: 1.2rem;
          font-weight: 300;
          margin-bottom: 10px;
      }
</style>