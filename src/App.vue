<script setup>
import { onBeforeMount, ref, toRaw } from 'vue';
import { PlusIcon as PlusIconMini } from '@heroicons/vue/solid';
import { TrashIcon } from '@heroicons/vue/outline';
import { Web5 } from '@tbd54566975/web5';

let web5;
let myDid;
let forms = [
  { name: '', artist: '' },
  { name: '', artist: '' },
  { name: '', artist: '' },
]
let tracks = ref([]);

onBeforeMount(async () => {
  ({ web5, did: myDid } = await Web5.connect());

  // Populate tracks from DWN
  const { records } = await web5.dwn.records.query({
    message: {
      filter: {
        schema: 'http://schema.org/MusicPlaylist'
      },
      dateSort: 'createdAscending'
    }
  });

  // Add entry to tracks array
  for (let record of records) {
    const data = await record.data.json();
    const track = { 
      record, 
      name: data.name, 
      artist: data.artist,
      image: data.image, 
      id: record.id 
    };
    tracks.value.push(track);
  }
});

// Fetch image
async function getImage(name, artist) {
  try {
    const response = await fetch(`https://ws.audioscrobbler.com/2.0/?method=track.getInfo&api_key=${import.meta.env.VITE_LAST_FM_KEY}&artist=${artist}&track=${name}&format=json`);
    const data = await response.json();
    if (
      data &&
      data.track &&
      data.track.album &&
      Array.isArray(data.track.album.image) &&
      data.track.album.image.length > 1
    ) {
      const imageURL = data.track.album.image[1]['#text'];
      return imageURL;
    } else {
      throw new Error("Image URL not found in the response.");
    }
  } catch (error) {
    console.error(error);
    // Return a default image URL when there is an error
    return new URL(`./assets/notes.jpeg`, import.meta.url).href;
  }
}

// Adding tracks
async function addTrack(form) {

  let imageURL = await getImage(form.name, form.artist)

  const trackData = {
    name: form.name,
    artist: form.artist,
    image: imageURL
  };

  form.name = ''
  form.artist = ''

  // Create the record in DWN
  const { record } = await web5.dwn.records.create({
    data    : trackData,
    message : {
      schema     : 'http://schema.org/MusicPlaylist',
      dataFormat : 'application/json'
    }
  });

  // add DWeb message recordId as a way to reference the message for further operations
  // e.g. updating it or overwriting it
  const data = await record.data.json();
  const track = { 
    record, 
    name: data.name,
    artist: data.artist, 
    image: data.image, 
    id: record.id 
  };
  tracks.value.push(track);
}

// Delete Track
async function deleteTrack(trackItem) {
  let deletedTrack;
  let index = 0;

  for (let track of tracks.value) {
    if (trackItem.id === track.id) {
      deletedTrack = track;
      break;
    }
    index ++;
  }

  tracks.value.splice(index, 1);

  // Delete the record in DWN
  await web5.dwn.records.delete({
    message: {
      recordId: deletedTrack.id
    }
  });
}

</script>

<template>
  <div class="flex flex-col items-center justify-center min-h-full px-8 py-12 sm:px-6">
    <!-- Title -->
    <div class="sm:max-w-md sm:w-full">
      <h2 class="font-bold text-3xl text-center tracking-tight">
        My Favorite Songs
      </h2>
    </div>
    <!-- Add track Form -->
    <div class="mt-16" v-for="(form, index) in forms" :key="index">
      <form class="flex space-x-4" @submit.prevent="addTrack(form, index)">
        <div class="border-b border-gray-200 sm:w-full">
          <label for="'add-track-' + index" class="sr-only">Add a title</label>
          <textarea
            rows="1" name="'add-track-' + index" id="'add-track-' + index" ref="titles" v-model="form.name"
            @keydown.enter.exact.prevent="addTrack(form, index)"
            class="block border-0 border-transparent focus:ring-0 p-0 pb-2 resize-none sm:text-sm w-96"
            placeholder="Title" />
          <label for="add-artist" class="sr-only">Add an artist</label>
          <textarea
            rows="1" name="'add-artist-' + index" id="'add-artist-' + index" ref="artists" v-model="form.artist"
            @keydown.enter.exact.prevent="addTrack(form, index)"
            class="block border-0 border-transparent focus:ring-0 p-0 pb-2 resize-none sm:text-sm w-96"
            placeholder="Artist" />
        </div>
        <button type="submit" class="bg-indigo-600 border border-transparent focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:ring-offset-2 hover:bg-indigo-700 inline-flex items-center p-1 rounded-full shadow-sm text-white">
          <PlusIconMini class="h-5 w-5" aria-hidden="true" />
        </button>
      </form>
    </div>
    <!-- Tracks -->
    <div v-if="(tracks.length > 0)" class="border-gray-200 border-t border-x mt-16 rounded-lg shadow-md sm:max-w-xl sm:mx-auto sm:w-full">
      <div v-for="track in tracks" :key="track" class="border-b border-gray-200 flex items-center p-4">
        <div>
          <img :src="(track.image)" alt="album image" width="70"/>
        </div>
        <div class="font-light ml-3 text-gray-500 text-xl">
          {{ track.name }} by {{ track.artist }}
        </div>
        <!-- Delete Track -->
        <div class="ml-auto">
          <div @click="deleteTrack(track)" class="cursor-pointer">
            <TrashIcon class="h-8 text-gray-200 w-8" :class="'text-red-500'" />
          </div>
        </div>
      </div>
    </div>
  </div>
</template>