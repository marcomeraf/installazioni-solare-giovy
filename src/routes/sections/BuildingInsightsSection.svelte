<!--
 Copyright 2023 Google LLC

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

      https://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
 -->

<script lang="ts">
  /* global google */

  import type { MdDialog } from '@material/web/dialog/dialog';
  import Expandable from '../components/Expandable.svelte';
  import {
    type BuildingInsightsResponse,
    type RequestError,
    findClosestBuilding,
    type SolarPanelConfig,
  } from '../solar';
  import Show from '../components/Show.svelte';
  import SummaryCard from '../components/SummaryCard.svelte';
  import { createPalette, normalize, rgbToColor } from '../visualize';
  import { panelsPalette } from '../colors';
  import InputBool from '../components/InputBool.svelte';
  import InputPanelsCount from '../components/InputPanelsCount.svelte';
  import { showNumber } from '../utils';
  import NumberInput from '../components/InputNumber.svelte';
  import Gauge from '../components/Gauge.svelte';

  export let expandedSection: string;
  export let buildingInsights: BuildingInsightsResponse | undefined;
  export let configId: number | undefined;
  export let panelCapacityWatts: number;
  export let showPanels: boolean;
  export let customArea: google.maps.Polygon | null;

  export let googleMapsApiKey: string;
  export let geometryLibrary: google.maps.GeometryLibrary;
  export let location: google.maps.LatLng;
  export let map: google.maps.Map;

  const icon = 'home';
  const title = 'Endpoint Informazioni Edificio';

  let requestSent = false;
  let requestError: RequestError | undefined;
  let apiResponseDialog: MdDialog;

  let panelConfig: SolarPanelConfig | undefined;
  $: if (buildingInsights && configId !== undefined) {
    panelConfig = buildingInsights.solarPotential.solarPanelConfigs[configId];
  }

  let solarPanels: google.maps.Polygon[] = [];
  $: solarPanels.map((panel, i) =>
    panel.setMap(showPanels && panelConfig && i < panelConfig.panelsCount ? map : null),
  );

  let panelCapacityRatio = 1.0;
  $: if (buildingInsights) {
    const defaultPanelCapacity = buildingInsights.solarPotential.panelCapacityWatts;
    panelCapacityRatio = panelCapacityWatts / defaultPanelCapacity;
  }

  function isPointInPolygon(point: google.maps.LatLng, polygon: google.maps.Polygon): boolean {
    return google.maps.geometry.poly.containsLocation(point, polygon);
  }

  export async function showSolarPotential(location: google.maps.LatLng) {
    if (requestSent) {
      return;
    }

    buildingInsights = undefined;
    requestError = undefined;

    solarPanels.map((panel) => panel.setMap(null));
    solarPanels = [];

    requestSent = true;
    try {
      buildingInsights = await findClosestBuilding(location, googleMapsApiKey);
    } catch (e) {
      requestError = e as RequestError;
      return;
    } finally {
      requestSent = false;
    }

    const solarPotential = buildingInsights.solarPotential;

    // Filter panels based on custom area if it exists
    const filteredPanels = customArea 
      ? solarPotential.solarPanels.filter(panel => 
          isPointInPolygon(
            new google.maps.LatLng(panel.center.latitude, panel.center.longitude),
            customArea
          )
        )
      : solarPotential.solarPanels;

    // Create visual panels on map
    const palette = createPalette(panelsPalette).map(rgbToColor);
    const minEnergy = filteredPanels.slice(-1)[0]?.yearlyEnergyDcKwh || 0;
    const maxEnergy = filteredPanels[0]?.yearlyEnergyDcKwh || 0;

    solarPanels = filteredPanels.map((panel) => {
      const [w, h] = [solarPotential.panelWidthMeters / 2, solarPotential.panelHeightMeters / 2];
      const points = [
        { x: +w, y: +h },
        { x: +w, y: -h },
        { x: -w, y: -h },
        { x: -w, y: +h },
        { x: +w, y: +h },
      ];
      const orientation = panel.orientation == 'PORTRAIT' ? 90 : 0;
      const azimuth = solarPotential.roofSegmentStats[panel.segmentIndex].azimuthDegrees;
      const colorIndex = Math.round(normalize(panel.yearlyEnergyDcKwh, maxEnergy, minEnergy) * 255);
      return new google.maps.Polygon({
        paths: points.map(({ x, y }) =>
          geometryLibrary.spherical.computeOffset(
            { lat: panel.center.latitude, lng: panel.center.longitude },
            Math.sqrt(x * x + y * y),
            Math.atan2(y, x) * (180 / Math.PI) + orientation + azimuth,
          ),
        ),
        strokeColor: '#B0BEC5',
        strokeOpacity: 0.9,
        strokeWeight: 1,
        fillColor: palette[colorIndex],
        fillOpacity: 0.9,
      });
    });

    // Update configurations based on filtered panels
    if (buildingInsights) {
      // Calculate energy ratio based on filtered panels
      const totalOriginalEnergy = solarPotential.solarPanels.reduce((sum, panel) => sum + panel.yearlyEnergyDcKwh, 0);
      const totalFilteredEnergy = filteredPanels.reduce((sum, panel) => sum + panel.yearlyEnergyDcKwh, 0);
      const energyRatio = totalFilteredEnergy / totalOriginalEnergy;

      // Update configurations
      buildingInsights.solarPotential.solarPanelConfigs = buildingInsights.solarPotential.solarPanelConfigs.map(config => ({
        ...config,
        panelsCount: Math.min(Math.round(config.panelsCount * (filteredPanels.length / solarPotential.solarPanels.length)), filteredPanels.length),
        yearlyEnergyDcKwh: config.yearlyEnergyDcKwh * energyRatio,
        roofSegmentSummaries: config.roofSegmentSummaries.map(summary => ({
          ...summary,
          panelsCount: Math.round(summary.panelsCount * (filteredPanels.length / solarPotential.solarPanels.length)),
          yearlyEnergyDcKwh: summary.yearlyEnergyDcKwh * energyRatio
        }))
      }));

      // Update other relevant statistics
      buildingInsights.solarPotential.maxArrayPanelsCount = filteredPanels.length;
      buildingInsights.solarPotential.maxArrayAreaMeters2 *= filteredPanels.length / solarPotential.solarPanels.length;
    }
  }

  $: showSolarPotential(location);
</script>

{#if requestError}
  <div class="error-container on-error-container-text">
    <Expandable section={title} icon="error" {title} subtitle={requestError.error.status}>
      <div class="grid place-items-center py-2 space-y-4">
        <div class="grid place-items-center">
          <p class="body-medium">
            Errore nella richiesta <code>buildingInsights</code>
          </p>
          <p class="title-large">ERRORE {requestError.error.code}</p>
          <p class="body-medium"><code>{requestError.error.status}</code></p>
          <p class="label-medium">{requestError.error.message}</p>
        </div>
        <md-filled-button role={undefined} on:click={() => showSolarPotential(location)}>
          Riprova
          <md-icon slot="icon">refresh</md-icon>
        </md-filled-button>
      </div>
    </Expandable>
  </div>
{:else if !buildingInsights}
  <div class="grid py-8 place-items-center">
    <md-circular-progress four-color indeterminate />
  </div>
{:else if configId !== undefined && panelConfig}
  <Expandable
    bind:section={expandedSection}
    {icon}
    {title}
    subtitle={`Energia annuale: ${(
      (panelConfig.yearlyEnergyDcKwh * panelCapacityRatio) /
      1000
    ).toFixed(2)} MWh`}
  >
    <div class="flex flex-col space-y-2 px-2">
      <span class="outline-text label-medium">
        <b>{title}</b> fornisce dati sulla posizione, dimensioni e potenziale solare di un edificio.
      </span>

      <InputPanelsCount
        bind:configId
        solarPanelConfigs={buildingInsights.solarPotential.solarPanelConfigs}
      />
      <NumberInput
        bind:value={panelCapacityWatts}
        icon="bolt"
        label="Capacità pannello"
        suffix="Watt"
      />
      <InputBool bind:value={showPanels} label="Pannelli solari" />

      <div class="grid justify-items-end">
        <md-filled-tonal-button role={undefined} on:click={() => apiResponseDialog.show()}>
          Risposta API
        </md-filled-tonal-button>
      </div>

      <md-dialog bind:this={apiResponseDialog}>
        <div slot="headline">
          <div class="flex items-center primary-text">
            <md-icon>{icon}</md-icon>
            <b>&nbsp;{title}</b>
          </div>
        </div>
        <div slot="content">
          <Show value={buildingInsights} label="buildingInsightsResponse" />
        </div>
        <div slot="actions">
          <md-text-button role={undefined} on:click={() => apiResponseDialog.close()}>
            Chiudi
          </md-text-button>
        </div>
      </md-dialog>
    </div>
  </Expandable>

  {#if expandedSection == title}
    <div class="absolute top-0 left-0 w-72">
      <div class="flex flex-col space-y-2 m-2">
        <SummaryCard
          {icon}
          {title}
          rows={[
            {
              icon: 'wb_sunny',
              name: 'Ore di sole annuali',
              value: showNumber(buildingInsights.solarPotential.maxSunshineHoursPerYear),
              units: 'ore',
            },
            {
              icon: 'square_foot',
              name: 'Area tetto',
              value: showNumber(buildingInsights.solarPotential.wholeRoofStats.areaMeters2),
              units: 'm²',
            },
            {
              icon: 'solar_power',
              name: 'Numero max pannelli',
              value: showNumber(buildingInsights.solarPotential.maxArrayPanelsCount),
              units: 'pannelli',
            },
            {
              icon: 'co2',
              name: 'Risparmio CO₂',
              value: showNumber(buildingInsights.solarPotential.carbonOffsetFactorKgPerMwh),
              units: 'Kg/MWh',
            },
          ]}
        />

        <div class="p-4 w-full surface on-surface-text rounded-lg shadow-md">
          <div class="flex justify-around">
            <Gauge
              icon="solar_power"
              title="Numero pannelli"
              label={showNumber(panelConfig.panelsCount)}
              labelSuffix={`/ ${showNumber(buildingInsights.solarPotential.maxArrayPanelsCount)}`}
              max={buildingInsights.solarPotential.maxArrayPanelsCount}
              value={panelConfig.panelsCount}
            />

            <Gauge
              icon="energy_savings_leaf"
              title="Energia annuale"
              label={showNumber((panelConfig?.yearlyEnergyDcKwh ?? 0) * panelCapacityRatio)}
              labelSuffix="KWh"
              max={buildingInsights.solarPotential.solarPanelConfigs.slice(-1)[0]
                .yearlyEnergyDcKwh * panelCapacityRatio}
              value={panelConfig.yearlyEnergyDcKwh * panelCapacityRatio}
            />
          </div>
        </div>
      </div>
    </div>
  {/if}
{/if}