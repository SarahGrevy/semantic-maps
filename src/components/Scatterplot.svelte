<script>
  import { onMount, onDestroy } from 'svelte';
  import DetailCard from './DetailCard.svelte';
  import { schemeCategory10 } from 'd3-scale-chromatic';
  import { scaleLinear, scaleOrdinal } from 'd3-scale';
  import { min, max } from 'd3-array';
  import reglConstructor from 'regl';

  export let data = [];
  export let domainColumn = "";
  export let opacity = 1;
  export let selectedValues = new Set();
  export let searchQuery = "";
  export let highlightedData = [];
  export let startDate = null;
  export let endDate = null;

  let canvas;
  let regl;
  let drawPoints;
  let hoveredData = null;
  let containerWidth = 800;
  let containerHeight = 600;
  const radius = 6;

  $: highlightedSet = new Set(highlightedData.map(d => d.id));
  $: xDomain = [min(data, d => d.x), max(data, d => d.x)];
  $: yDomain = [min(data, d => d.y), max(data, d => d.y)];
  $: xScale = scaleLinear()
    .domain(xDomain[0] === xDomain[1] ? [xDomain[0] - 1, xDomain[1] + 1] : xDomain)
    .range([0, containerWidth]);
  $: yScale = scaleLinear()
    .domain(yDomain[0] === yDomain[1] ? [yDomain[0] - 1, yDomain[1] + 1] : yDomain)
    .range([containerHeight, 0]);
  $: colorScale = scaleOrdinal()
    .domain([...new Set(data.map(d => d[domainColumn]))])
    .range(schemeCategory10);

  // --- OPTIMIZED: Use Typed Arrays for WebGL attributes ---
  let positions = null;
  let colors = null;

  $: if (data.length) {
    // First, count how many points will be drawn (all points, but selected last)
    // We'll use two passes: non-selected, then selected
    const nonSelected = [];
    const selected = [];

    // Precompute min/max for date only once
    const dateTimes = data.map(d => d.date.getTime());
    const minDate = Math.min(...dateTimes);
    const maxDate = Math.max(...dateTimes);

    for (let i = 0; i < data.length; ++i) {
      const d = data[i];
      const matchesSearch = searchQuery && d.title?.toLowerCase().includes(searchQuery.toLowerCase());
      const isHighlighted = highlightedSet.has(d.id) || matchesSearch;
      const isSelected = selectedValues.has(d[domainColumn]);
      const isInDateRange = (!startDate || d.date >= startDate) && (!endDate || d.date <= endDate);
      const isFullDateRange = startDate?.getTime() === minDate && endDate?.getTime() === maxDate;

      let alpha = opacity;
      if ((isHighlighted || isSelected) && isInDateRange && !isFullDateRange) {
        alpha = 1;
      } else if (isInDateRange && !isFullDateRange && selectedValues.size === 0) {
        alpha = 1;
      } else if ((isHighlighted || isSelected) && isFullDateRange) {
        alpha = 1;
      }

      const color = colorScale(d[domainColumn]) || '#1f77b4';
      let rgb;
      if (color.startsWith('#')) {
        rgb = [
          parseInt(color.slice(1, 3), 16) / 255,
          parseInt(color.slice(3, 5), 16) / 255,
          parseInt(color.slice(5, 7), 16) / 255
        ];
      } else if (color.startsWith('rgb')) {
        const nums = color.match(/\d+/g).map(Number);
        rgb = [nums[0] / 255, nums[1] / 255, nums[2] / 255];
      } else {
        rgb = [31 / 255, 119 / 255, 180 / 255];
      }

      const point = {
        x: 2 * xScale(d.x) / containerWidth - 1,
        y: 1 - 2 * yScale(d.y) / containerHeight,
        rgb,
        alpha
      };

      // If selected/highlighted, push to selected, else nonSelected
      if (alpha === 1) {
        selected.push(point);
      } else {
        nonSelected.push(point);
      }
    }

    // Combine: non-selected first, then selected
    const allPoints = [...selected, ...nonSelected];
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
      count: positions.length / 2 // or allPoints.length
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
          if (r > 0.5) discard;
          gl_FragColor = fragColor;
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

  // --- OPTIMIZED: Throttle hover handler ---
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
      // Use screen coordinates for hover
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
  }

  // function handleMouseLeave() {
  //   hoveredData = null;
  // }
</script>

<div class="chart-container">
  <canvas
    bind:this={canvas}
    width={containerWidth}
    height={containerHeight}
    style="width: {containerWidth}px; height: {containerHeight}px; display: block;"
    on:mousemove={handleMouseMove}
  />
  {#if hoveredData}
    <DetailCard {hoveredData} {data} {domainColumn} {colorScale} />
  {/if}
</div>

<style>
  .chart-container {
    position: relative;
    display: flex;
    justify-content: center;
    align-items: center;
    flex-direction: column;
    width: 100%;
    height: 100%;
  }
  canvas {
    cursor: crosshair;
    border-radius: 8px;
    background-color: white;
    max-width: 100%;
    height: auto;
  }
</style>