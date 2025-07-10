<script>
  import { onMount, onDestroy } from 'svelte';
  import DetailCard from './DetailCard.svelte';
  import { scaleLinear } from 'd3-scale';
  import { min, max } from 'd3-array';
  import reglConstructor from 'regl';

  export let data = [];
  export let domainColumn = "";
  export const opacity = 0.6;
  export const selectedValues = new Set();
  export const searchQuery = "";
  export let highlightedData = [];
  export const startDate = null;
  export const endDate = null;

  let canvas;
  let regl;
  let drawPoints;
  let hoveredData = null;
  let containerWidth = 800;
  let containerHeight = 600;
  const radius = 6;
  let hoveredIndex = -1;

  let selectedStatus = null;

  function toggleStatus(status) {
    selectedStatus = selectedStatus === status ? null : status;
  }

  $: highlightedSet = new Set(highlightedData.map(d => d.id));
  $: xDomain = [min(data, d => d.x), max(data, d => d.x)];
  $: yDomain = [min(data, d => d.y), max(data, d => d.y)];

  const xExtent = [
    min(data, d => d.x) - 15,// Adjusted for zooming in or out 
    max(data, d => d.x) + 15
  ];
  const yExtent = [
    min(data, d => d.y) - 20,
    max(data, d => d.y) + 20
  ];

  $: xScale = scaleLinear().domain(xExtent).range([0, containerWidth]);
  $: yScale = scaleLinear().domain(yExtent).range([containerHeight, 0]);

  function getStatusColor(status) {
    switch (status) {
      case 'CURRENTLY_RATED_HELPFUL': return '#2ca02c';
      case 'CURRENTLY_RATED_NOT_HELPFUL': return '#d62728';
      case 'NEEDS_MORE_RATINGS': return '#808080';
      default: return '#1f77b4';
    }
  }

  let positions = null;
  let colors = null;

  $: if (data.length) {
    const visibleData = selectedStatus
      ? data.filter(d => d.currentStatus === selectedStatus)
      : data;

    const allPoints = visibleData.map(d => {
      const color = getStatusColor(d.currentStatus);
      let rgb;
      if (color.startsWith('#')) {
        rgb = [
          parseInt(color.slice(1, 3), 16) / 255,
          parseInt(color.slice(3, 5), 16) / 255,
          parseInt(color.slice(5, 7), 16) / 255
        ];
      } else {
        rgb = [31 / 255, 119 / 255, 180 / 255];
      }

      return {
        x: 2 * xScale(d.x) / containerWidth - 1,
        y: 1 - 2 * yScale(d.y) / containerHeight,
        rgb,
        alpha: d.currentStatus === 'NEEDS_MORE_RATINGS' ? 0.6 : 0.8
      };
    });

    positions = new Float32Array(allPoints.length * 2);
    colors = new Float32Array(allPoints.length * 4);

    for (let i = 0; i < allPoints.length; ++i) {
      const p = allPoints[i];
      positions[2 * i] = p.x;
      positions[2 * i + 1] = p.y;
      colors[4 * i] = p.rgb[0];
      colors[4 * i + 1] = p.rgb[1];
      colors[4 * i + 2] = p.rgb[2];
      colors[4 * i + 3] = p.alpha;
    }
  }

  function drawWebGL() {
    if (!regl || !positions || !colors) return;
    regl.clear({ color: [1, 1, 1, 1], depth: 1 });
    drawPoints({
      position: positions,
      color: colors,
      count: positions.length / 2
    });
  }

  onMount(() => {
    regl = reglConstructor({ canvas });

    drawPoints = regl({
      vert: `
        precision mediump float;
        attribute vec2 position;
        attribute vec4 color;
        varying vec4 fragColor;
        void main() {
          fragColor = color;
          gl_Position = vec4(position, 0, 1);
          gl_PointSize = ${radius * 2}.0;
        }
      `,
      frag: `
        precision mediump float;
        varying vec4 fragColor;
        void main() {
          float r = length(gl_PointCoord - 0.5);
          if (r > 0.5) {
            discard;
          } else {
            gl_FragColor = fragColor;
          }
        }
      `,
      attributes: {
        position: { buffer: regl.prop('position'), size: 2 },
        color: { buffer: regl.prop('color'), size: 4 }
      },
      count: regl.prop('count'),
      primitive: 'points'
    });

    drawWebGL();
  });

  $: if (regl && positions && colors) {
    drawWebGL();
  }

  onDestroy(() => {
    if (regl) regl.destroy();
  });

  let lastHover = 0;
  function handleMouseMove(event) {
    const now = performance.now();
    if (now - lastHover < 30) return;
    lastHover = now;

    const rect = canvas.getBoundingClientRect();
    const mouseX = (event.clientX - rect.left) * (containerWidth / rect.width);
    const mouseY = (event.clientY - rect.top) * (containerHeight / rect.height);

    let found = null;
    let minDist = radius + 3;
    for (let i = 0; i < data.length; ++i) {
      const px = xScale(data[i].x);
      const py = yScale(data[i].y);
      const dx = px - mouseX;
      const dy = py - mouseY;
      const dist = Math.sqrt(dx * dx + dy * dy);
      if (dist < minDist) {
        found = data[i];
        minDist = dist;
      }
    }

    hoveredData = found;
    canvas.style.cursor = found?.url ? 'pointer' : 'crosshair';
  }

  function handleClick(event) {
    const rect = canvas.getBoundingClientRect();
    const mouseX = (event.clientX - rect.left) * (containerWidth / rect.width);
    const mouseY = (event.clientY - rect.top) * (containerHeight / rect.height);

    let closest = null;
    let minDist = radius + 3;
    for (let i = 0; i < data.length; ++i) {
      const px = xScale(data[i].x);
      const py = yScale(data[i].y);
      const dx = px - mouseX;
      const dy = py - mouseY;
      const dist = Math.sqrt(dx * dx + dy * dy);
      if (dist < minDist) {
        closest = data[i];
        minDist = dist;
      }
    }

    if (closest && closest.url) {
      window.open(closest.url, '_blank');
    }
  }
</script>

<div class="chart-container">
  <div class="legend">
    <div class="legend-title">✨ Filter by status of Notes:</div>
    <div class="legend-items">
      <div
        class="legend-item {selectedStatus === 'CURRENTLY_RATED_HELPFUL' ? 'active' : ''}"
        on:click={() => toggleStatus('CURRENTLY_RATED_HELPFUL')}
      >
        <span class="legend-color" style="background-color: #2ca02c;"></span> Helpful
      </div>
      <div
        class="legend-item {selectedStatus === 'CURRENTLY_RATED_NOT_HELPFUL' ? 'active' : ''}"
        on:click={() => toggleStatus('CURRENTLY_RATED_NOT_HELPFUL')}
      >
        <span class="legend-color" style="background-color: #d62728;"></span> Not Helpful
      </div>
      <div
        class="legend-item {selectedStatus === 'NEEDS_MORE_RATINGS' ? 'active' : ''}"
        on:click={() => toggleStatus('NEEDS_MORE_RATINGS')}
      >
        <span class="legend-color" style="background-color: #808080;"></span> Needs More Ratings
      </div>
    </div>
    <br>
    <div class="legend-subtitle">*Click on a dot on the map to open the tweet URL for the associated tweet.
</div>
  </div>

  <canvas
    bind:this={canvas}
    width={containerWidth}
    height={containerHeight}
    on:click={handleClick}
    on:mousemove={handleMouseMove}
    style="width: {containerWidth}px; height: {containerHeight}px; display: block;"
  />

  {#if hoveredData}
    <DetailCard {hoveredData} {data} {domainColumn} {getStatusColor} />
  {/if}
</div>

<style>
  .chart-container {
    position: relative;
    width: 100%;
    padding-bottom: 50%;
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  canvas {
    cursor: crosshair;
    border-radius: 8px;
    background-color: white;
    max-width: 100%;
    height: auto;
  }

  .legend {
    position: absolute;
    top: -35px;
    left: 0px;
    background: white;
    border-radius: 8px;
    padding: 10px 12px;
    box-shadow: 0 2px 6px rgba(0,0,0,0.15);
    font-size: 14px;
    z-index: 10;
  }

  .legend-title {
    font-weight: bold;
    margin-bottom: 6px;
  }

   .legend-subtitle {
    font-weight: italic;
  }

  .legend-items {
    display: flex;
    gap: 10px;
    flex-wrap: wrap;
  }

  .legend-item {
    display: flex;
    align-items: center;
    gap: 5px;
    cursor: pointer;
    padding: 3px 6px;
    border-radius: 4px;
    transition: background 0.2s;
  }

  .legend-item:hover {
    background-color: #f0f0f0;
  }

  .legend-item.active {
    background-color: #dfefff;
    font-weight: bold;
  }

  .legend-color {
    width: 12px;
    height: 12px;
    border-radius: 50%;
  }
</style>
