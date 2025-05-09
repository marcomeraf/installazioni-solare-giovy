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

  import * as GMAPILoader from '@googlemaps/js-api-loader';
  const { Loader } = GMAPILoader;

  import { onMount } from 'svelte';

  import SearchBar from './components/SearchBar.svelte';
  import Sections from './sections/Sections.svelte';

  const googleMapsApiKey = import.meta.env.VITE_GOOGLE_MAPS_API_KEY;
  const defaultPlace = {
    name: 'Biblioteca Rinconada',
    address: '1213 Newell Rd, Palo Alto, CA 94303',
  };
  let location: google.maps.LatLng | undefined;
  const zoom = 19;

  // Initialize app.
  let mapElement: HTMLElement;
  let map: google.maps.Map;
  let geometryLibrary: google.maps.GeometryLibrary;
  let mapsLibrary: google.maps.MapsLibrary;
  let placesLibrary: google.maps.PlacesLibrary;
  onMount(async () => {
    // Load the Google Maps libraries.
    const loader = new Loader({ apiKey: googleMapsApiKey });
    const libraries = {
      geometry: loader.importLibrary('geometry'),
      maps: loader.importLibrary('maps'),
      places: loader.importLibrary('places'),
    };
    geometryLibrary = await libraries.geometry;
    mapsLibrary = await libraries.maps;
    placesLibrary = await libraries.places;

    // Get the address information for the default location.
    const geocoder = new google.maps.Geocoder();
    const geocoderResponse = await geocoder.geocode({
      address: defaultPlace.address,
    });
    const geocoderResult = geocoderResponse.results[0];

    // Initialize the map at the desired location.
    location = geocoderResult.geometry.location;
    map = new mapsLibrary.Map(mapElement, {
      center: location,
      zoom: zoom,
      tilt: 0,
      mapTypeId: 'satellite',
      mapTypeControl: false,
      fullscreenControl: false,
      rotateControl: false,
      streetViewControl: false,
      zoomControl: false,
    });
  });
</script>

<svelte:head>
  <title>Demo API Solare</title>
  <meta name="description" content="Demo API Solare" />
</svelte:head>

<!-- Barra superiore -->
<div class="flex flex-row h-full">
  <!-- Mappa principale -->
  <div bind:this={mapElement} class="w-full" />

  <!-- Barra laterale -->
  <aside class="flex-none md:w-96 w-80 p-2 pt-3 overflow-auto">
    <div class="flex flex-col space-y-2 h-full">
      {#if placesLibrary && map}
        <SearchBar bind:location {placesLibrary} {map} initialValue={defaultPlace.name} />
      {/if}

      <div class="p-4 surface-variant outline-text rounded-lg space-y-3">
        <p>
          <a
            class="primary-text"
            href="https://developers.google.com/maps/documentation/solar/overview?hl=it"
            target="_blank"
          >
            Due endpoint distinti dell'<b>API Solare</b>
            <md-icon class="text-sm">open_in_new</md-icon>
          </a>
          offrono molti vantaggi ai siti web di mercato solare, agli installatori solari e ai designer di SaaS solari.
        </p>

        <p>
          <b>Clicca su un'area qui sotto</b>
          per vedere che tipo di informazioni può fornire l'API Solare.
        </p>
      </div>

      {#if location}
        <Sections {location} {map} {geometryLibrary} {googleMapsApiKey} />
      {/if}

      <div class="grow" />

      <div class="flex flex-col items-center w-full">
        <md-text-button
          href="https://github.com/googlemaps-samples/js-solar-potential"
          target="_blank"
        >
          Visualizza codice su GitHub
          <img slot="icon" src="github-mark.svg" alt="GitHub" width="16" height="16" />
        </md-text-button>
      </div>

      <span class="pb-4 text-center outline-text label-small">
        Questo non è un prodotto ufficialmente supportato da Google.
      </span>
    </div>
  </aside>
</div>