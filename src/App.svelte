<script>
  import * as topojson from "topojson-client";
  import { geoPath } from "d3-geo";
  import { geoConicConformalSpain } from "d3-composite-projections";
  import { draw } from "svelte/transition";
  import municipalities from "$data/municipalities.json";
  import puertos from "$data/puertos.js";
  import { scaleLinear } from "d3-scale";
  import { max, min } from "d3-array";
  import Tooltip from "$components/Tooltip.svelte";

  let provincias = topojson.feature(
    municipalities,
    municipalities.objects.provinces,
  ).features;
  let bordes = topojson.mesh(
    municipalities,
    municipalities.objects.autonomous_regions,
    (a, b) => a !== b,
  );

  $: compositionBorders = projection.getCompositionBorders();

  let width = 400;
  $: height = width * 0.7;

  $: projection = geoConicConformalSpain()
    .scale(width > 1200 ? 4000 : width * 3.5)
    .translate(width > 1200 ? [width / 2, 400] : [width / 2, height / 2]);
  $: path = geoPath(projection);

  let puertoType = "Mercancias";

  function setPuertoType(type) {
    puertoType = type;
  }

  const colorRanges = {
    Mercancias: ["#8dcbef", "#1b8bce"],
    Pesca: ["#8ce5f0", "#1db3ce"],
    Pasajeros: ["#8cf0e6", "#14c4b2"],
  };

  const borderRanges = {
    Mercancias: ["#5a8299", "#105680"],
    Pesca: ["#599199", "#0d6f80"],
    Pasajeros: ["#599993", "#0d8074"],
  };

  $: sizeScale = scaleLinear()
    .domain([0, max(puertos, (d) => d[puertoType])])
    .range([2, width < 668 ? 25 : 50]);

  $: colorScale = scaleLinear()
    .domain([0, max(puertos, (d) => d[puertoType])])
    .range(colorRanges[puertoType]);

  $: colorBorderScale = scaleLinear()
    .domain([0, max(puertos, (d) => d[puertoType])])
    .range(borderRanges[puertoType]);

  let selection;

  window.addEventListener("DOMContentLoaded", (event) => {
    function updateIframeHeight() {
      const el = document.documentElement;
      const rect = el.getBoundingClientRect();
      const styles = window.getComputedStyle(el);
      const margin =
        parseFloat(styles.marginTop) + parseFloat(styles.marginBottom);
      const height = Math.ceil(rect.height + margin);

      window.parent.postMessage(
        {
          type: "resize-iframe",
          value: height,
        },
        "*",
      );
    }
    updateIframeHeight();

    if (window.ResizeObserver) {
      new ResizeObserver(() => {
        updateIframeHeight();
      }).observe(document.documentElement);
    } else {
      window.addEventListener("load", updateIframeHeight);
      window.addEventListener("resize", updateIframeHeight);
    }

    window.addEventListener(
      "message",
      (event) => {
        if (event.data.type === "request-resize") {
          updateIframeHeight();
        }
      },
      false,
    );
  });
</script>

<div class="button-container">
  <button
    on:click={() => setPuertoType("Mercancias")}
    class:selected={puertoType === "Mercancias"}
    style="background-color: #1b8bce">Mercancías</button
  >
  <button
    on:click={() => setPuertoType("Pesca")}
    class:selected={puertoType === "Pesca"}
    style="background-color: #1db3ce">Pesca</button
  >
  <button
    on:click={() => setPuertoType("Pasajeros")}
    class:selected={puertoType === "Pasajeros"}
    style="background-color: #14c4b2">Pasajeros</button
  >
</div>

<div class="fuente">
  <p>Datos provisionales de 2023 (Puertos del Estado)</p>
</div>

<!-- svelte-ignore a11y-click-events-have-key-events -->
<div
  class="chart-container"
  bind:clientWidth={width}
  on:click={() => {
    selection = null;
  }}
>
  <svg {width} {height} on:click|stopPropagation>
    <g class="legend">
      <circle
        cx={width / 5}
        cy={height / 2.5}
        r={sizeScale(min(puertos, (d) => d[puertoType]))}
        fill="none"
        stroke={colorBorderScale(min(puertos, (d) => d[puertoType]))}
        stroke-width="0.8"
      />
      <text
        class="legend-text"
        x={width / 5 - 1}
        y={height / 2.5 + sizeScale(min(puertos, (d) => d[puertoType])) + 12}
        text-anchor="middle"
      >
        {puertoType === "Mercancias"
          ? "91.000"
          : puertoType === "Pesca"
            ? "5"
            : "170"}
      </text>
      <circle
        cx={width / 5}
        cy={height / 2.5}
        r={sizeScale(max(puertos, (d) => d[puertoType]))}
        fill="none"
        stroke={colorBorderScale(max(puertos, (d) => d[puertoType]))}
        stroke-width="0.8"
      />
      <text
        class="legend-text"
        y={height / 2.5 + sizeScale(max(puertos, (d) => d[puertoType])) + 13}
        text-anchor="middle"
      >
        <tspan x={width / 5 - 1}
          >{(
            Math.round(max(puertos, (d) => d[puertoType]) / 10000) * 10000
          ).toLocaleString("es-ES")}</tspan
        >
        <tspan x={width / 5 - 1} dy="1em">
          {puertoType === "Mercancias" || puertoType === "Pesca"
            ? " toneladas"
            : " pasajeros"}
        </tspan>
      </text>
    </g>
    {#each provincias as provincia}
      <!-- svelte-ignore a11y-click-events-have-key-events -->
      <path
        d={path(provincia)}
        fill="#EBEBEB"
        stroke="black"
        stroke-width="0.05"
      />
    {/each}

    <path d={path(bordes)} fill="none" stroke="black" stroke-width="0.1" />
    <path
      d={compositionBorders}
      fill="none"
      stroke="black"
      stroke-width="0.5"
    />

    {#each puertos as puerto}
      {#if puerto[puertoType] >= 1}
        <!-- svelte-ignore a11y-no-noninteractive-tabindex -->
        <!-- svelte-ignore a11y-mouse-events-have-key-events -->
        <circle
          cx={projection([puerto.Longitude, puerto.Latitude])[0]}
          cy={projection([puerto.Longitude, puerto.Latitude])[1]}
          r={sizeScale(puerto[puertoType])}
          fill={colorScale(puerto[puertoType])}
          fill-opacity={selection
            ? selection === puerto
              ? "100%"
              : "30%"
            : "70%"}
          stroke={colorBorderScale(puerto[puertoType])}
          stroke-width="0.8"
          stroke-opacity={selection
            ? selection === puerto
              ? "100%"
              : "70%"
            : "100%"}
          on:mouseover={() => {
            selection = puerto;
          }}
          on:focus={() => {
            selection = puerto;
          }}
          tabindex="0"
          on:mouseout={() => {
            selection = null;
          }}
        />
      {/if}
    {/each}
  </svg>

  {#if selection}
    <Tooltip
      {projection}
      {selection}
      {width}
      {height}
      {puertoType}
      {colorRanges}
    />
  {/if}
</div>

<style>
  .chart-container {
    max-height: 810px;
  }

  svg {
    max-height: 810px;
  }

  circle {
    outline: none;
  }

  .button-container {
    display: flex;
    justify-content: center;
    margin-bottom: 8px;
  }

  .button-container button {
    margin: 0 3px;
    padding: 4px 8px;
    font-size: 14px;
    cursor: pointer;
    color: white;
    border: none;
    border-radius: 8px;
  }

  .button-container button:hover {
    opacity: 70%;
  }

  .button-container button:active {
    background-color: #004085;
  }

  .button-container button.selected {
    font-weight: bold;
    box-shadow: 0 0 0 1px black;
  }

  .fuente {
    font-style: italic;
    font-size: 12px;
    text-align: center;
    padding: 0px 0px 10px;
  }

  .legend-text {
    font-style: italic;
    font-size: 11px;
  }
</style>
