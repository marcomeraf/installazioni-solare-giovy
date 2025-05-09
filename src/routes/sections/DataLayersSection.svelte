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

  import { onMount } from 'svelte';

  import type { MdDialog } from '@material/web/dialog/dialog';
  import Calendar from '../components/Calendar.svelte';
  import Dropdown from '../components/Dropdown.svelte';
  import Expandable from '../components/Expandable.svelte';
  import { getLayer, type Layer } from '../layer';
  import {
    getDataLayerUrls,
    type BuildingInsightsResponse,
    type DataLayersResponse,
    type LayerId,
    type RequestError,
  } from '../solar';
  import InputBool from '../components/InputBool.svelte';
  import Show from '../components/Show.svelte';
  import SummaryCard from '../components/SummaryCard.svelte';
  import type { MdSlider } from '@material/web/slider/slider';
  import InputNumber from '../components/InputNumber.svelte';

  export let expandedSection: string;
  export let showPanels = true;

  export let googleMapsApiKey: string;
  export let buildingInsights: BuildingInsightsResponse | undefined;
  export let geometryLibrary: google.maps.GeometryLibrary;
  export let map: google.maps.Map;
  export let location: google.maps.LatLng;

  const icon = 'layers';
  const title = 'Endpoint Data Layers';

  const dataLayerOptions: Record<LayerId | 'none', string> = {
    none: 'Nessun layer',
    mask: 'Maschera tetto',
    dsm: 'Modello superficie digitale',
    rgb: 'Immagine aerea',
    annualFlux: 'Irraggiamento annuale',
    monthlyFlux: 'Irraggiamento mensile',
    hourlyShade: 'Ombreggiatura oraria',
  };

  const monthNames = [
    'Gen',
    'Feb',
    'Mar',
    'Apr',
    'Mag',
    'Giu',
    'Lug',
    'Ago',
    'Set',
    'Ott',
    'Nov',
    'Dic',
  ];

  let dataLayersResponse: DataLayersResponse | undefined;
  let requestError: RequestError | undefined;
  let apiResponseDialog: MdDialog;
  let layerId: LayerId | 'none' = 'monthlyFlux';
  let layer: Layer | undefined;
  let imageryQuality: 'HIGH' | 'MEDIUM' | 'LOW';

  let playAnimation = true;
  let tick = 0;
  let month = 0;
  let day = 14;
  let hour = 0;

  let radius = 50;
  let overlays: google.maps.GroundOverlay[] = [];
  let showRoofOnly = false;
  
  async function showDataLayer(reset = false) {
    if (reset) {
      dataLayersResponse = undefined;
      requestError = undefined;
      layer = undefined;

      showRoofOnly = ['annualFlux', 'monthlyFlux', 'hourlyShade'].includes(layerId);
      map.setMapTypeId(layerId == 'rgb' ? 'roadmap' : 'satellite');
      overlays.map((overlay) => overlay.setMap(null));
      month = layerId == 'hourlyShade' ? 3 : 0;
      day = 14;
      hour = 5;
      playAnimation = ['monthlyFlux', 'hourlyShade'].includes(layerId);
    }
    if (layerId == 'none') {
      return;
    }

    if (!layer) {
      try {
        dataLayersResponse = await getDataLayerUrls(
          { latitude: location.lat(), longitude: location.lng() },
          radius,
          googleMapsApiKey
        );
      } catch (e) {
        requestError = e as RequestError;
        return;
      }

      imageryQuality = dataLayersResponse.imageryQuality;

      try {
        layer = await getLayer(layerId, dataLayersResponse, googleMapsApiKey);
      } catch (e) {
        requestError = e as RequestError;
        return;
      }
    }

    const bounds = layer.bounds;
    console.log('Render layer:', {
      layerId: layer.id,
      showRoofOnly: showRoofOnly,
      month: month,
      day: day,
    });
    overlays.map((overlay) => overlay.setMap(null));
    overlays = layer
      .render(showRoofOnly, month, day)
      .map((canvas) => new google.maps.GroundOverlay(canvas.toDataURL(), bounds));

    if (!['monthlyFlux', 'hourlyShade'].includes(layer.id)) {
      overlays[0].setMap(map);
    }
  }

  $: if (layer?.id == 'monthlyFlux') {
    overlays.map((overlay, i) => overlay.setMap(i == month ? map : null));
  } else if (layer?.id == 'hourlyShade') {
    overlays.map((overlay, i) => overlay.setMap(i == hour ? map : null));
  }

  function onSliderChange(event: Event) {
    const target = event.target as MdSlider;
    if (layer?.id == 'monthlyFlux') {
      if (target.valueStart != month) {
        month = target.valueStart ?? 0;
      } else if (target.valueEnd != month) {
        month = target.valueEnd ?? 0;
      }
      tick = month;
    } else if (layer?.id == 'hourlyShade') {
      if (target.valueStart != hour) {
        hour = target.valueStart ?? 0;
      } else if (target.valueEnd != hour) {
        hour = target.valueEnd ?? 0;
      }
      tick = hour;
    }
  }

  $: if (layer?.id == 'monthlyFlux') {
    if (playAnimation) {
      month = tick % 12;
    } else {
      tick = month;
    }
  } else if (layer?.id == 'hourlyShade') {
    if (playAnimation) {
      hour = tick % 24;
    } else {
      tick = hour;
    }
  }

  onMount(() => {
    showDataLayer(true);

    setInterval(() => {
      tick++;
    }, 1000);
  });
</script>

{#if requestError}
  <div class="error-container on-error-container-text">
    <Expandable section={title} icon="error" {title} subtitle={requestError.error.status}>
      <div class="grid place-items-center py-2 space-y-4">
        <div class="grid place-items-center">
          <p class="body-medium">
            Errore nella richiesta <code>dataLayers</code>
            {layerId}
          </p>
          <p class="title-large">ERRORE {requestError.error.code}</p>
          <p class="body-medium"><code>{requestError.error.status}</code></p>
          <p class="label-medium">{requestError.error.message}</p>
        </div>
        <md-filled-button role={undefined} on:click={() => showDataLayer(true)}>
          Riprova
          <md-icon slot="icon">refresh</md-icon>
        </md-filled-button>
      </div>
    </Expandable>
  </div>
{:else}
  <Expandable bind:section={expandedSection} {icon} {title} subtitle={dataLayerOptions[layerId]}>
    <div class="flex flex-col space-y-2 px-2">
      <span class="outline-text label-medium">
        <b>{title}</b> fornisce immagini grezze ed elaborate e dettagli granulari su un'area circostante una posizione.
      </span>

      <InputNumber
        bind:value={radius}
        icon="radio_button_checked"
        label="Raggio area"
        suffix="m"
        onChange={() => showDataLayer(true)}
      />

      <Dropdown
        bind:value={layerId}
        options={dataLayerOptions}
        onChange={async () => {
          layer = undefined;
          showDataLayer();
        }}
      />

      {#if layerId == 'none'}
        <div />
      {:else if !layer}
        <md-linear-progress four-color indeterminate />
      {:else}
        {#if layer.id == 'hourlyShade'}
          <Calendar bind:month bind:day onChange={async () => showDataLayer()} />
        {/if}

        <span class="outline-text label-medium">
          {#if imageryQuality == 'HIGH'}
            <p><b>Immagini aeree a bassa quota</b> disponibili.</p>
            <p>Immagini e dati DSM elaborati a <b>10 cm/pixel</b>.</p>
          {:else if imageryQuality == 'MEDIUM'}
            <p><b>Immagini aeree aumentate con AI</b> disponibili.</p>
            <p>Immagini e dati DSM elaborati a <b>25 cm/pixel</b>.</p>
          {:else if imageryQuality == 'LOW'}
            <p><b>Immagini aeree o satellitari aumentate con AI</b> disponibili.</p>
            <p>Immagini e dati DSM elaborati a <b>50 cm/pixel</b>.</p>
          {/if}
        </span>

        <InputBool bind:value={showPanels} label="Pannelli solari" />
        <InputBool bind:value={showRoofOnly} label="Solo tetto" onChange={() => showDataLayer()} />

        {#if ['monthlyFlux', 'hourlyShade'].includes(layerId)}
          <InputBool bind:value={playAnimation} label="Riproduci animazione" />
        {/if}
      {/if}
      <div class="flex flex-row">
        <div class="grow" />
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
          <Show value={dataLayersResponse} label="dataLayersResponse" />
        </div>
        <div slot="actions">
          <md-text-button role={undefined} on:click={() => apiResponseDialog.close()}>
            Chiudi
          </md-text-button>
        </div>
      </md-dialog>
    </div>
  </Expandable>
{/if}

<div class="absolute top-0 left-0 w-72">
  {#if expandedSection == title && layer}
    <div class="m-2">
      <SummaryCard {icon} {title} rows={[{ name: dataLayerOptions[layerId], value: '' }]}>
        <div class="flex flex-col space-y-4">
          <p class="outline-text">
            {#if layerId == 'mask'}
              L'immagine della maschera dell'edificio: un bit per pixel che indica se quel pixel è considerato parte di un tetto o meno.
            {:else if layerId == 'dsm'}
              Un'immagine del DSM (Modello Digitale della Superficie) della regione. I valori sono in metri sopra il geoide EGM96 (cioè, livello del mare). Le posizioni non valide (dove non abbiamo dati) sono memorizzate come -9999.
            {:else if layerId == 'rgb'}
              Un'immagine dei dati RGB (foto aerea) della regione.
            {:else if layerId == 'annualFlux'}
              La mappa del flusso annuale (luce solare annuale sui tetti) della regione. I valori sono kWh/kW/anno. Questo è un flusso non mascherato: il flusso viene calcolato per ogni posizione, non solo per i tetti degli edifici. Le posizioni non valide sono memorizzate come -9999: le posizioni al di fuori della nostra area di copertura saranno non valide, e alcune posizioni all'interno dell'area di copertura, dove non siamo stati in grado di calcolare il flusso, saranno anch'esse non valide.
            {:else if layerId == 'monthlyFlux'}
              La mappa del flusso mensile (luce solare sui tetti, suddivisa per mese) della regione. I valori sono kWh/kW/anno. Il file di immagine GeoTIFF puntato da questo URL conterrà dodici bande, corrispondenti a Gennaio...Dicembre, in ordine.
            {:else if layerId == 'hourlyShade'}
              Dodici URL per l'ombreggiatura oraria, corrispondenti a Gennaio...Dicembre, in ordine. Ogni file di immagine GeoTIFF conterrà 24 bande, corrispondenti alle 24 ore del giorno. Ogni pixel è un intero a 32 bit, corrispondente ai (fino a) 31 giorni di quel mese; un bit 1 significa che la posizione corrispondente è in grado di vedere il sole in quel giorno, di quell'ora, di quel mese. Le posizioni non valide sono memorizzate come -9999 (poiché questo è negativo, ha il bit 31 impostato, e nessun valore valido potrebbe avere il bit 31 impostato poiché corrisponderebbe al 32° giorno del mese).
            {/if}
          </p>

          {#if layer.palette}
            <div>
              <div
                class="h-2 outline rounded-sm"
                style={`background: linear-gradient(to right, ${layer.palette.colors.map(
                  (hex) => '#' + hex,
                )})`}
              />
              <div class="flex justify-between pt-1 label-small">
                <span>{layer.palette.min}</span>
                <span>{layer.palette.max}</span>
              </div>
            </div>
          {/if}
        </div>
      </SummaryCard>
    </div>
  {/if}
</div>

<div class="absolute bottom-6 left-0 w-full">
  <div class="md:mr-96 mr-80 grid place-items-center">
    {#if layer}
      <div
        class="flex items-center surface on-surface-text pr-4 text-center label-large rounded-full shadow-md"
      >
        {#if layer.id == 'monthlyFlux'}
          <md-slider
            range
            min={0}
            max={11}
            value-start={month}
            value-end={month}
            on:input={onSliderChange}
          />
          <span class="w-8">{monthNames[month]}</span>
        {:else if layer.id == 'hourlyShade'}
          <md-slider
            range
            min={0}
            max={23}
            value-start={hour}
            value-end={hour}
            on:input={onSliderChange}
          />
          <span class="w-24 whitespace-nowrap">
            {monthNames[month]}
            {day},
            {#if hour == 0}
              12am
            {:else if hour < 10}
              {hour}am
            {:else if hour < 12}
              {hour}am
            {:else if hour == 12}
              12pm
            {:else if hour < 22}
              {hour - 12}pm
            {:else}
              {hour - 12}pm
            {/if}
          </span>
        {/if}
      </div>
    {/if}
  </div>
</div>