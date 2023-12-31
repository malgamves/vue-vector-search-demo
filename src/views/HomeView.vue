<script setup lang="ts">
import weaviate, {
  WeaviateClient,
  ObjectsBatcher,
  ApiKey,
} from "weaviate-ts-client";
import { onMounted, ref } from "vue";
import { useStorage } from "@vueuse/core";

const results = ref();
const step = ref(1);
const searchQuery = ref("");

let stepState = useStorage("steps-local-storage", step.value);

const client: WeaviateClient = weaviate.client({
  scheme: "https",
  host: import.meta.env.VITE_WEAVIATE_HOST_URL,
  apiKey: new ApiKey(import.meta.env.VITE_WEAVIATE_API_KEY),
  headers: {
    "X-OpenAI-Api-Key": import.meta.env.VITE_OPENAI_KEY,
  },
});

const classObj = {
  class: "Music50",
  vectorizer: "text2vec-openai", 
  moduleConfig: {
    "text2vec-openai": {},
    "generative-openai": {}, 
  },
};

async function addSchema() {
  const res = await client.schema.classCreator().withClass(classObj).do();
  // console.log(res);
}

async function getJsonData() {
  const file = await fetch(
    "https://gist.githubusercontent.com/malgamves/ca7d28dae20672912c85c358d7e42e31/raw/c323dd7e1de78073d772df870b4056e1c3e52a1b/songs.json"
  );
  return file.json();
}

async function importMusic() {
  // Get the questions directly from the URL
  const data = await getJsonData();

  // Prepare a batcher
  let batcher: ObjectsBatcher = client.batch.objectsBatcher();
  let counter = 0;
  const batchSize = 100;

  for (const songs of data) {
    // Construct an object with a class and properties corresponding to data shape
    const obj = {
      class: "Music50",
      properties: {
        rank: songs.rank,
        title: songs.title,
        artist: songs.artist,
        album: songs.album,
        year: songs.year,
      },
    };

    // add the object to the batch queue
    batcher = batcher.withObject(obj);

    // When the batch counter reaches batchSize, push the objects to Weaviate
    if (counter++ == batchSize) {
      // flush the batch queue
      const res = await batcher.do();
      console.log(res);

      // restart the batch queue
      counter = 0;
      batcher = client.batch.objectsBatcher();
    }
  }

  // Flush the remaining objects
  const res = await batcher.do();
  console.log(res);
}

async function nearTextQuery() {
  const res = await client.graphql
    .get()
    .withClassName("Music50")
    .withFields("title artist rank album year")
    .withNearText({ concepts: [`"${searchQuery.value}"`] })
    .withLimit(2)
    .do();

  results.value = res.data.Get.Music50;
  return res;
}

async function generativeSearchQuery() {
  const res = await client.graphql
    .get()
    .withClassName("Music50")
    .withFields("title artist album rank year")
    .withNearText({ concepts: [`"${searchQuery.value}"`] })
    .withGenerate({
      singlePrompt: "Give me a fun fact about the song {title} by {artist} in one short paragraph.",
    })
    .withLimit(2)
    .do();
  
  results.value = res.data.Get.Music50;
  return res;
}

onMounted(async () => {});

function nextStep() {
  if (stepState != 3) {
    stepState.value = stepState.value + 1;
  }
}
</script>

<template>
  <main>
    <h1 class="text-5xl pt-5 font-bold text-teal-500 text-center">
      vector search demo
    </h1>

    <div class="items-center pb-20 pt-10 p-8">
      <h2 class="sr-only">Steps</h2>

      <div>
        <div class="overflow-hidden rounded-full bg-gray-200">
          <div
            class="h-2 rounded-full bg-teal-400"
            :class="`${'w-' + stepState + '/3'}`"
          ></div>
        </div>

        <ol class="mt-4 grid grid-cols-3 text-sm font-medium text-gray-500">
          <li class="flex items-center justify-start text-teal-400 sm:gap-1.5">
            <span class="hidden sm:inline"> 1. Initialize </span>
          </li>

          <li class="flex items-center justify-center sm:gap-1.5">
            <span
              class="hidden sm:inline"
              :class="`${stepState >= 2 ? 'text-teal-400' : ' text-gray-500'}`"
            >
              2. Import Data
            </span>
          </li>

          <li class="flex items-center justify-end sm:gap-1.5">
            <span
              class="hidden sm:inline"
              :class="`${stepState == 3 ? 'text-teal-400' : ' text-gray-500'}`"
            >
              3. Search
            </span>
          </li>
        </ol>
      </div>

      <div
        class="items-center justify-center flex-col flex pt-8"
        :class="`${stepState == 3 ? 'hidden' : ''}`"
      >
        <button
          class="inline-block items-center rounded-full border border-teal-600 bg-teal-600 p-3 text-white hover:bg-transparent hover:text-teal-600 focus:outline-none focus:ring active:text-teal-500"
          @click="nextStep()"
        >
          <span class="sr-only"> Download </span>

          <svg
            class="h-5 w-5 rtl:rotate-180"
            xmlns="http://www.w3.org/2000/svg"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M14 5l7 7m0 0l-7 7m7-7H3"
            />
          </svg>
        </button>
        <p class="text-xs pt-2 text-gray-500">next step</p>
      </div>
    </div>

    <section v-if="stepState == 1">
      <div class="mx-auto px-4 pt-6 sm:px-6 lg:pt-8">
        <div class="p-6 md:pt-8 lg:pt-8">
          <div class="mx-auto max-w-xl text-center">
            <h2 class="text-2xl font-bold text-teal-500 md:text-3xl">
              we've initialised weaviate for you
            </h2>

            <p class="hidden text-teal-500 sm:mt-4 sm:block">
              all you have to do now is add a schema to your weaviate instance
              by clicking the button below
            </p>
            <div class="mt-4 md:mt-8">
              <button
                @click="addSchema()"
                class="inline-block rounded border border-slate-600 bg-slate-600 px-12 py-3 text-sm font-medium text-teal-500 transition hover:bg-transparent hover:text-teal-500 focus:outline-none focus:ring focus:ring-yellow-400"
              >
                add schema
              </button>
            </div>
          </div>
        </div>
      </div>
    </section>

    <section v-if="stepState == 2">
      <div class="mx-auto px-4 pt-6 sm:px-6 lg:pt-8">
        <div class="p-6 md:pt-8 lg:pt-8">
          <div class="mx-auto max-w-xl text-center">
            <h2 class="text-2xl font-bold text-teal-500 md:text-3xl">
              time to import some data
            </h2>

            <p class="hidden text-teal-500 sm:mt-4 sm:block">
              we're importing rolling stones top 500 songs of the last century,
              once imported you can search it
            </p>
            <div class="mt-4 md:mt-8">
              <button
                @click="importMusic()"
                class="inline-block rounded border border-slate-600 bg-slate-600 px-12 py-3 text-sm font-medium text-teal-500 transition hover:bg-transparent hover:text-teal-500 focus:outline-none focus:ring focus:ring-yellow-400"
              >
                import data
              </button>
            </div>
          </div>
        </div>
      </div>
    </section>

    <div v-if="stepState == 3" class="justify-center items-center">
      <div class="flex items-center justify-center pt-3">
        <label for="Search" class="sr-only"> Search </label>

        <input
          v-model="searchQuery"
          type="text"
          id="Search"
          placeholder="Search for..."
          class="w-2/3 rounded-md border-gray-200 bg-slate-300 py-2.5 pl-4 pe-10 shadow-sm sm:text-sm"
        />

      </div>

      <div class="flex items-start justify-center pt-5 gap-2">

        <button
                @click="nearTextQuery()"
                class="inline-block rounded border border-white bg-slate-600 px-12 py-3 text-sm font-medium text-teal-500 transition hover:bg-transparent hover:text-teal-500 focus:outline-none focus:ring focus:ring-yellow-400"
              >
                Search
              </button>


        <button
                @click="generativeSearchQuery()"
                class="inline-block rounded border border-white bg-slate-600 px-12 py-3 text-sm font-medium text-teal-500 transition hover:bg-transparent hover:text-teal-500 focus:outline-none focus:ring focus:ring-yellow-400"
              >
                Generative Search
              </button>
      </div>

      <div v-if="results" class="flex items-start justify-center">
        <div
          class="item-start absolute flex z-10 mt-2 w-[750px] divide-y divide-gray-100 rounded-md border border-gray-100 bg-white shadow-lg"
          role="menu"
        >
          <div class="p-2">
            <strong
              class="block p-2 text-xs font-medium uppercase text-gray-400"
            >
              results
            </strong>

            <div v-for="result in results">
              <div class="space-y-4">
                <details class="group [&_summary::-webkit-details-marker]:hidden" open>
                  <summary
                    class="flex cursor-pointer items-center justify-between gap-1.5 rounded-lg text-sm p-4 text-gray-900"
                  >
                    <h2 class="font-medium">
                      {{ result.title }} by {{ result.artist }} from
                {{ result.album }} rank {{ result.rank }}
                    </h2>
              
                    <svg
                      v-if="result._additional"
                      class="h-5 w-5 shrink-0 transition duration-300 group-open:-rotate-180"
                      xmlns="http://www.w3.org/2000/svg"
                      fill="none"
                      viewBox="0 0 24 24"
                      stroke="currentColor"
                    >
                      <path
                        stroke-linecap="round"
                        stroke-linejoin="round"
                        stroke-width="2"
                        d="M19 9l-7 7-7-7"
                      />
                    </svg>
                  </summary>
              
                  <p 
                  v-if="result._additional"
                  class="mt-2 px-4 text-xs leading-relaxed text-gray-700">
                    {{ result._additional.generate.singleResult }}
                  </p>
                </details>
            
              </div>
            </div>

            
          </div>

          
        </div>
        
      </div>
    </div>

  </main>
</template>
