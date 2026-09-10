<script>
  import { onMount } from "svelte";
  import arrow from "$lib/assets/arrow_right.svg";
  let observations = $state([]);
  let range = $state(3);
  let listLength = $state(30);
  const getObservations = async (latitude, longitude, radius, length) => {
    let date = new Date();
    date.setMonth(date.getMonth() - 1);
    let created = `${date.getFullYear()}-${date.getMonth() + 1}-${date.getDate()}`;
    let request = await fetch(
      `https://api.inaturalist.org/v2/observations?created_d1=${created}&lat=${latitude}&lng=${longitude}&radius=${radius}&per_page=${length}&order=desc&order_by=observed_on&fields=(photos:(url:!t),species_guess:!t,observed_on:!t,taxon:(id:!t,name:!t))`,
    );
    let data = await request.json();
    console.log(data.results);

    for (let entry of data.results) {
      let name = entry.species_guess;
      if (!name || name === "") {
        continue;
      }
      let photo = entry.photos[0].url.replace("square", "medium");

      let observedOn = entry.observed_on;
      observations.push({
        id: entry.uuid,
        name,
        photo,
        observedOn,
        checked: false,
        taxon: entry.taxon,
      });
    }
    localStorage.setItem("observations", JSON.stringify(observations));
  };
  const getLatLong = async () => {
    if (navigator.geolocation) {
      let position = navigator.geolocation.getCurrentPosition(
        (position) => {
          let lat = position.coords.latitude;
          let lng = position.coords.longitude;
          getObservations(lat, lng, range, listLength);
          return position;
        },
        () => {
          console.log("Unable to get geolocation");
          return null;
        },
      );
      return position;
    }
  };
  onMount(async () => {
    let items = localStorage.getItem("observations");

    if (items) {
      console.log(items);
      observations = JSON.parse(items);
    } else {
      getLatLong();
    }
  });
</script>

<div class="header">
  <h1>iNaturalist Scavenger Hunt</h1>
  <div class="row">
    <label
      >Number of Specimens:<input
        type="number"
        bind:value={listLength}
      /></label
    >
    <label>Search Range: <input type="number" bind:value={range} /></label>
  </div>
  <button
    onclick={() => {
      observations = [];
      localStorage.removeItem("observations");
      getLatLong();
    }}>Start a New Search</button
  >
</div>
<div class="list">
  {#each observations as observation}
    <div
      class="entry"
      class:checked={observation.checked}
      onclick={() => {
        let foundObservation = observations.findIndex(
          (entry) => entry.id === observation.id,
        );
        observations[foundObservation].checked =
          !observations[foundObservation].checked;
        localStorage.setItem("observations", JSON.stringify(observations));
      }}
    >
      <img src={observation.photo} />
      <div class="text">
        <h2>{observation.name}</h2>
        {#if observation?.taxon}
          <a
            href={`https://www.inaturalist.org/taxa/${observation?.taxon.id}`}
            target="_blank"
            >{observation?.taxon.name}<img src={arrow} />
          </a>
        {/if}
      </div>
    </div>
  {/each}
  <div class="progress">
    {observations.filter((entry) => entry.checked).length} of {observations.length}
    found
  </div>
</div>

<style>
  :global(body) {
    background-color: #f1f7ed;
    color: #122c34;
  }
  .header {
    font-family: sans-serif;
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: space-between;
    .row {
      display: flex;
      flex-direction: row;
      gap: 2rem;
      label {
        display: flex;
        flex-direction: column;
      }
    }
    button {
      background-color: #1d84b5;
      border: none;
      border-radius: 10px;
      color: #f1f7ed;
      padding: 0.5rem;
    }
  }
  .list {
    display: flex;
    flex-wrap: wrap;
    gap: 1rem;
    margin-bottom: 5rem;
    .entry {
      background-color: #7ca982;
      width: 300px;
      border-radius: 20px;
      display: flex;
      flex-direction: column;

      border: 1px solid #92828d;
      filter: drop-shadow(0 3px 3px #92828d);
      .text {
        padding-left: 20px;
        padding-right: 20px;
      }
      a {
        display: flex;
        flex-direction: row;
        align-items: center;
        gap: 0.1rem;
        text-decoration: none;
        color: #122c34;
        img {
          width: 18px;
          height: 18px;
        }
      }
      img {
        object-fit: cover;
        border-radius: 20px 20px 0 0;
      }
      &.checked {
        background-color: #92828d;
        img {
          filter: grayscale(1);
        }
        filter: none;
      }
      h2 {
        font-weight: normal;
        font-family: sans-serif;
      }
    }
  }
  .progress {
    position: fixed;
    display: flex;
    justify-content: center;
    align-items: center;

    bottom: 0;
    left: 0;
    width: 100vw;
    padding: 1rem;
    background-color: #1d84b5;
    color: #f1f7ed;
    filter: drop-shadow(0 0 10px #92828d);
  }
</style>
