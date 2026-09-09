<script>
  import { onMount } from "svelte";

  let observations = $state([]);
  const getObservations = async (latitude, longitude, radius) => {
    let date = new Date();
    date.setMonth(date.getMonth() - 1);
    let created = `${date.getFullYear()}-${date.getMonth() + 1}-${date.getDate()}`;
    let request = await fetch(
      `https://api.inaturalist.org/v2/observations?created_d1=${created}&lat=${latitude}&lng=${longitude}&radius=${radius}&per_page=20&order=desc&order_by=observed_on&fields=(photos:(url:!t),species_guess:!t,observed_on:!t)`,
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
          getObservations(lat, lng, 3);
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
      <h2>{observation.name}</h2>
    </div>
  {/each}
</div>

<style>
  .header {
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: space-between;
    button {
      height: 50px;
    }
  }
  .list {
    display: flex;
    flex-wrap: wrap;
    gap: 1rem;
    .entry {
      width: 300px;
      border-radius: 20px;
      display: flex;
      flex-direction: column;
      img {
        object-fit: cover;
        border-radius: 20px 20px 0 0;
      }
      &.checked {
        background-color: lightgray;
      }
      h2 {
        padding-left: 20px;
        padding-right: 20px;
        font-weight: normal;
        font-family: sans-serif;
      }
      border: 1px solid gray;
    }
  }
</style>
