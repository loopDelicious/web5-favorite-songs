<script setup>
import { onBeforeMount, ref, toRaw } from 'vue';
import { PlusIcon as PlusIconMini } from '@heroicons/vue/solid';
import { CheckCircleIcon, TrashIcon } from '@heroicons/vue/outline';
import { Web5 } from '@tbd54566975/web5';

let web5;
let myDid;
const songs = ref([]);

onBeforeMount(async () => {
  ({ web5, did: myDid } = await Web5.connect());

  // Populate songs from DWN
  const { records } = await web5.dwn.records.query({
    message: {
      filter: {
        schema: 'http://schema.org/MusicPlaylist'
      },
      dateSort: 'createdAscending'
    }
  });

  // Add entry to songs array
  for (let record of records) {
    const data = await record.data.json();
    const song = { 
      record, 
      title: data.title, 
      artist: data.artist, 
      image: data.image, 
      id: record.id 
    };
    songs.value.push(song);
  }

  // Provide a fallback album art image
  // const imgUrl = new URL(`./assets/notes.jpeg`, import.meta.url).href;
});

// // Function to fetch the image and convert it to base64
// async function fetchAndEncodeImageToBase64(url) {
//   try {
//     const response = await fetch(url);
//     if (!response.ok) {
//       throw new Error(`Failed to fetch the image. Status: ${response.status} ${response.statusText}`);
//     }

//     const imageBlob = await response.blob();
//     const reader = new FileReader();

//     // Convert the image Blob to base64
//     reader.onloadend = () => {
//       const base64String = reader.result.split(',')[1];
//       console.log(base64String); // Base64 encoded image data
//     };

//     reader.readAsDataURL(imageBlob);
//   } catch (error) {
//     console.error(error);
//   }
// }

// Fetch image
async function getImage(title, artist) {
  try {
    const response = await fetch(`http://ws.audioscrobbler.com/2.0/?method=track.getInfo&api_key=${import.meta.env.VITE_LAST_FM_KEY}&artist=${artist}&track=${title}&format=json`);
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

// Adding songs
const newTitle = ref('');
const newArtist= ref('');

async function addSong() {

  let imageURL = await getImage(newTitle.value, newArtist.value)

  const songData = {
    title: newTitle.value,
    artist: newArtist.value,
    image: imageURL
  };

  newTitle.value = '';
  newArtist.value = '';

  // Create the record in DWN
  const { record } = await web5.dwn.records.create({
    data    : songData,
    message : {
      schema     : 'http://schema.org/MusicPlaylist',
      dataFormat : 'application/json'
    }
  });

  // add DWeb message recordId as a way to reference the message for further operations
  // e.g. updating it or overwriting it
  const data = await record.data.json();
  const song = { 
    record, 
    title: data.title, 
    artist: data.artist, 
    image: data.image, 
    id: record.id 
  };
  songs.value.push(song);
}

// Delete Song
async function deleteSong(songItem) {
  let deletedSong;
  let index = 0;

  for (let song of songs.value) {
    if (songItem.id === song.id) {
      deletedSong = song;
      break;
    }
    index ++;
  }

  songs.value.splice(index, 1);

  // Delete the record in DWN
  await web5.dwn.records.delete({
    message: {
      recordId: deletedSong.id
    }
  });
}

// // Toggling ToDo Status
// async function toggleTodoComplete(todoItem) {
//   let toggledTodo;
//   let updatedTodoData;

//   for (let todo of todos.value) {
//     if (todoItem.id === todo.id) {
//       toggledTodo = todo;
//       todo.data.completed = !todo.data.completed;
//       updatedTodoData = { ...toRaw(todo.data) };
//       break;
//     }
//   }

//   // Get record in DWN
//   const { record } = await web5.dwn.records.read({
//     message: {
//       recordId: toggledTodo.id,
//     }
//   });

//   // Update the record in DWN
//   await record.update({ data: updatedTodoData });
// }

</script>

<template>
  <div class="flex flex-col items-center justify-center min-h-full px-8 py-12 sm:px-6">
    <!-- Title -->
    <div class="sm:max-w-md sm:w-full">
      <h2 class="font-bold text-3xl text-center tracking-tight">
        My Favorite Songs
      </h2>
    </div>

    <!-- Add song Form -->
    <div class="mt-16">
      <form class="flex space-x-4" @submit.prevent="addSong">
        <div class="border-b border-gray-200 sm:w-full">
          <label for="add-song" class="sr-only">Add a title</label>
          <textarea
            rows="1" name="add-song" id="add-song" v-model="newTitle"
            @keydown.enter.exact.prevent="addSong"
            class="block border-0 border-transparent focus:ring-0 p-0 pb-2 resize-none sm:text-sm w-96"
            placeholder="Title" />
          <label for="add-artist" class="sr-only">Add an artist</label>
          <textarea
            rows="1" name="add-artist" id="add-artist" v-model="newArtist"
            @keydown.enter.exact.prevent="addSong"
            class="block border-0 border-transparent focus:ring-0 p-0 pb-2 resize-none sm:text-sm w-96"
            placeholder="Artist" />
        </div>
        <button type="submit" class="bg-indigo-600 border border-transparent focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:ring-offset-2 hover:bg-indigo-700 inline-flex items-center p-1 rounded-full shadow-sm text-white">
          <PlusIconMini class="h-5 w-5" aria-hidden="true" />
        </button>
      </form>
    </div>
    <!-- Songs -->
    <div v-if="(songs.length > 0)" class="border-gray-200 border-t border-x mt-16 rounded-lg shadow-md sm:max-w-xl sm:mx-auto sm:w-full">
      <div v-for="song in songs" :key="song" class="border-b border-gray-200 flex items-center p-4">
        <!-- <div @click="toggleTodoComplete(todo)" class="cursor-pointer">
          <CheckCircleIcon class="h-8 text-gray-200 w-8" :class="{ 'text-green-500': todo.data.completed }" />
        </div> -->
        <div>
          <img :src="(song.image)" alt="album image" width="70"/>
        </div>
        <div class="font-light ml-3 text-gray-500 text-xl">
          {{ song.title }} by {{ song.artist }}
        </div>
        <!-- Delete Song -->
        <div class="ml-auto">
          <div @click="deleteSong(song)" class="cursor-pointer">
            <TrashIcon class="h-8 text-gray-200 w-8" :class="'text-red-500'" />
          </div>
        </div>
      </div>
    </div>
  </div>
</template>