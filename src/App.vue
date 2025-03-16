<template>
  <div
    class="min-h-screen bg-black flex flex-col items-center justify-center p-4 font-montserrat"
  >
    <!-- Model Loading Message -->
    <div
      v-if="isModelLoading"
      class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50"
    >
      <div class="bg-white p-6 rounded-lg shadow-lg text-center">
        <svg
          class="animate-spin h-8 w-8 mx-auto mb-4 text-purple-500"
          viewBox="0 0 24 24"
        >
          <circle
            class="opacity-25"
            cx="12"
            cy="12"
            r="10"
            stroke="currentColor"
            stroke-width="4"
          ></circle>
          <path
            class="opacity-75"
            fill="currentColor"
            d="M4 12a8 8 0 018-8v8h8a8 8 0 01-8 8 8 8 0 01-8-8z"
          ></path>
        </svg>
        <p class="text-gray-700">Loading mood detection models...</p>
        <p class="text-sm text-gray-500">
          This may take a few seconds on first use.
        </p>
      </div>
    </div>

    <!-- Landing/State Management -->
    <div v-if="!photo && !result && !cameraActive" class="text-center">
      <h1 class="text-4xl font-bold mb-4 text-white">
        MoodSnap Movie Quote Generator
      </h1>
      <p class="mb-6 text-white">Capture your mood and get a movie quote!</p>
      <button
        @click="startCamera"
        class="bg-blue-500 text-white px-6 py-3 rounded-lg shadow hover:bg-blue-600 transition"
      >
        Take a Photo
      </button>
      <input
        type="file"
        accept="image/*"
        @change="uploadPhoto"
        class="hidden"
        ref="fileInput"
      />
      <button
        @click="$refs.fileInput.click()"
        class="bg-green-500 text-white px-6 py-3 rounded-lg shadow ml-4 hover:bg-green-600 transition"
      >
        Upload Photo
      </button>
    </div>

    <!-- Camera Preview -->
    <div v-if="cameraActive" class="flex flex-col items-center">
      <video
        ref="video"
        autoplay
        class="w-full max-w-96 rounded-lg shadow-lg mb-4"
        style="max-height: 600px"
      ></video>
      <button
        @click="capturePhoto"
        class="bg-blue-500 text-white px-6 py-3 rounded-lg shadow hover:bg-blue-600 transition"
      >
        Capture
      </button>
      <button
        @click="stopCamera"
        class="bg-red-500 text-white px-6 py-3 rounded-lg shadow mt-2 hover:bg-red-600 transition"
      >
        Cancel
      </button>
    </div>

    <!-- Photo Preview with Loading -->
    <div v-if="photo && !result" class="flex flex-col items-center">
      <img
        :src="photo"
        class="w-full max-w-96 rounded-lg shadow-lg mb-4 object-contain"
        alt="Captured Photo"
      />
      <button
        @click="analyzeMood"
        class="bg-purple-500 text-white px-6 py-3 rounded-lg shadow hover:bg-purple-600 transition flex items-center"
        :disabled="isLoading"
      >
        <span v-if="!isLoading">Analyze Mood</span>
        <span v-else class="flex items-center">
          <svg class="animate-spin h-5 w-5 mr-2" viewBox="0 0 24 24">
            <circle
              class="opacity-25"
              cx="12"
              cy="12"
              r="10"
              stroke="currentColor"
              stroke-width="4"
            ></circle>
            <path
              class="opacity-75"
              fill="currentColor"
              d="M4 12a8 8 0 018-8v8h8a8 8 0 01-8 8 8 8 0 01-8-8z"
            ></path>
          </svg>
          Analyzing...
        </span>
      </button>
      <button
        @click="reset"
        class="bg-gray-500 text-white px-6 py-3 rounded-lg shadow mt-2 hover:bg-gray-600 transition"
      >
        Try Again
      </button>
    </div>

    <!-- Result Screen -->
    <div v-if="result" class="flex flex-col items-center text-center">
      <div
        ref="resultImage"
        id="resultImage"
        class="relative rounded-lg shadow-lg overflow-hidden"
      >
        <img
          v-if="photo"
          :src="photo"
          class="w-full h-full object-cover"
          alt="Result"
        />
        <div class="absolute bottom-0 left-0 right-0 flex flex-col text-white">
          <div
            :style="{ backgroundColor: randomQuoteColor }"
            class="p-4 leading-tight"
          >
            <span class="font-bold text-lg">"{{ result.quote }}"</span><br />
            <span v-if="result.translation" class="text-sm italic">{{
              result.translation
            }}</span>
          </div>
          <div
            :style="{ backgroundColor: randomCharacterColor }"
            class="p-2 leading-tight"
          >
            <span class="text-xs"
              >{{ result.movieOriginal }} - {{ result.characterOriginal }}</span
            >
            <span
              v-if="
                result.movieOriginal != result.movie &&
                result.characterOriginal != result.character
              "
              class="text-xs italic"
              ><br />{{ result.movie }} - {{ result.character }}</span
            >
          </div>
        </div>
      </div>
      <p class="mt-4 text-white">
        Mood Detected:
        <span class="font-semibold capitalize">{{ result.mood }}</span>
      </p>
      <button
        @click="downloadImage"
        class="bg-green-500 text-white px-6 py-3 rounded-lg shadow mt-4 hover:bg-green-600 transition"
      >
        Download
      </button>
      <button
        @click="reset"
        class="bg-gray-500 text-white px-6 py-3 rounded-lg shadow mt-2 hover:bg-gray-600 transition"
      >
        Try Again
      </button>
    </div>

    <!-- No Face Detected Modal -->
    <NoFaceModal
      ref="noFaceModal"
      @retry="reset"
      @cancel="assignNeutralQuote"
    />
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import html2canvas from "html2canvas";
import Human from "@vladmandic/human";
import { quotes } from "./quotes";
import NoFaceModal from "./components/NoFaceModal.vue";

// State
const photo = ref(null);
const cameraActive = ref(false);
const video = ref(null);
const result = ref(null);
const fileInput = ref(null);
const isLoading = ref(false);
const isModelLoading = ref(false);
const imageWidth = ref(400); // Optimal width for 2:3
const imageHeight = ref(600); // Optimal height for 2:3
const screenWidth = ref(window.innerWidth - 32); // Account for p-4 padding
const OPTIMAL_WIDTH = 400;
const OPTIMAL_HEIGHT = 600;
const noFaceModal = ref(null); // Reference to modal
const randomQuoteColor = ref(null);
const randomCharacterColor = ref(null);

// Random background and overlay colors
const colors = ["#CD5700", "#0066CC", "#0343DF", "#EA27C2"];
const getRandomDistinctColors = () => {
  const shuffled = [...colors].sort(() => 0.5 - Math.random()); // Shuffle array
  return [shuffled[0] + "CC", shuffled[1] + "E6"]; // Take first two distinct colors with 80% and 90% opacity
};
[randomQuoteColor.value, randomCharacterColor.value] =
  getRandomDistinctColors();

// Update screenWidth on resize
onMounted(() => {
  initHuman();
  window.addEventListener("resize", () => {
    screenWidth.value = window.innerWidth - 32;
  });
});

// Function to crop and scale image to optimal dimensions (400x600)
const processImage = (sourceWidth, sourceHeight, drawCallback) => {
  const canvas = document.createElement("canvas");
  canvas.width = OPTIMAL_WIDTH;
  canvas.height = OPTIMAL_HEIGHT;
  const ctx = canvas.getContext("2d");

  const srcAspect = sourceWidth / sourceHeight;
  const targetAspect = OPTIMAL_WIDTH / OPTIMAL_HEIGHT;

  let srcWidth, srcHeight, srcX, srcY;
  if (srcAspect > targetAspect) {
    srcHeight = sourceHeight;
    srcWidth = sourceHeight * targetAspect;
    srcX = (sourceWidth - srcWidth) / 2;
    srcY = 0;
  } else {
    srcWidth = sourceWidth;
    srcHeight = sourceWidth / targetAspect;
    srcX = 0;
    srcY = (sourceHeight - srcHeight) / 2;
  }

  drawCallback(ctx, srcX, srcY, srcWidth, srcHeight);
  console.log("Processed image:", {
    width: canvas.width,
    height: canvas.height,
  });
  return canvas.toDataURL("image/png");
};

const humanConfig = {
  backend: "webgl",
  modelBasePath: "https://cdn.jsdelivr.net/npm/@vladmandic/human/models/",
  face: {
    enabled: true,
    detector: { rotation: false },
    emotion: { enabled: true },
  },
  body: { enabled: false },
  hand: { enabled: false },
  gesture: { enabled: false },
};
const human = new Human(humanConfig);

// init Human
const initHuman = async () => {
  isModelLoading.value = true;
  await human.load();
  console.log("Human models loaded:", human.models);
  isModelLoading.value = false;
};

// Camera Functions
const startCamera = async () => {
  cameraActive.value = true;
  const stream = await navigator.mediaDevices.getUserMedia({ video: true });
  video.value.srcObject = stream;
};

const capturePhoto = () => {
  const sourceWidth = video.value.videoWidth;
  const sourceHeight = video.value.videoHeight;
  photo.value = processImage(
    sourceWidth,
    sourceHeight,
    (ctx, srcX, srcY, srcWidth, srcHeight) => {
      ctx.drawImage(
        video.value,
        srcX,
        srcY,
        srcWidth,
        srcHeight,
        0,
        0,
        OPTIMAL_WIDTH,
        OPTIMAL_HEIGHT
      );
    }
  );
  console.log("Captured photo:", photo.value.slice(0, 50)); // Log start of base64 string
  imageWidth.value = OPTIMAL_WIDTH;
  imageHeight.value = OPTIMAL_HEIGHT;
  stopCamera();
};

const stopCamera = () => {
  const stream = video.value.srcObject;
  if (stream) {
    stream.getTracks().forEach((track) => track.stop());
  }
  cameraActive.value = false;
};

// Upload Photo
const uploadPhoto = (event) => {
  const file = event.target.files[0];
  if (file) {
    const reader = new FileReader();
    reader.onload = (e) => {
      const img = new Image();
      img.src = e.target.result;
      img.onload = () => {
        photo.value = processImage(
          img.width,
          img.height,
          (ctx, srcX, srcY, srcWidth, srcHeight) => {
            ctx.drawImage(
              img,
              srcX,
              srcY,
              srcWidth,
              srcHeight,
              0,
              0,
              OPTIMAL_WIDTH,
              OPTIMAL_HEIGHT
            );
          }
        );
        console.log("Uploaded photo:", photo.value.slice(0, 50)); // Log start of base64 string
        imageWidth.value = OPTIMAL_WIDTH;
        imageHeight.value = OPTIMAL_HEIGHT;
      };
    };
    reader.readAsDataURL(file);
  }
};

// Assign Neutral Quote
const assignNeutralQuote = () => {
  result.value = {
    mood: "neutral",
    ...quotes.neutral[Math.floor(Math.random() * quotes.neutral.length)],
  };
  isLoading.value = false;
};

// Mood Analysis with Human
const analyzeMood = async () => {
  isLoading.value = true;
  const img = new Image();
  img.src = photo.value;
  img.onload = async () => {
    const detections = await human.detect(img);
    const face = detections.face[0];
    if (!face) {
      console.log("No face detected");
      noFaceModal.value.open();
    } else {
      const expressions = face.emotion;
      console.log("Emotions: ", expressions);
      const mood = expressions.sort((a, b) =>
        a.score > b.score ? a.score : b.score
      )[0].emotion;
      const moodQuotes = quotes[mood] || quotes.neutral;
      const randomQuote =
        moodQuotes[Math.floor(Math.random() * moodQuotes.length)];
      result.value = { mood, ...randomQuote };
      isLoading.value = false;
    }
  };
};

// Image Download
const resultImage = ref(null);
const downloadImage = async () => {
  console.log("Downloading image...");
  const canvas = await html2canvas(resultImage.value, {
    width: OPTIMAL_WIDTH,
    height: OPTIMAL_HEIGHT,
    scale: 1,
    backgroundColor: null,
    onclone: function (clonedDoc) {
      // set the image to the desired width and height
      // where the display on screen is bounded by the container
      clonedDoc.getElementById(
        "resultImage"
      ).style.width = `${imageWidth.value}px`;
      clonedDoc.getElementById(
        "resultImage"
      ).style.height = `${imageHeight.value}px`;
    },
  });
  const link = document.createElement("a");
  link.download = `MoodSnap_${result.value.mood}_${Date.now()}.png`;
  link.href = canvas.toDataURL("image/png");
  console.log("Download URL:", link.href.slice(0, 50)); // Log start of base64 string
  link.click();
};

// Reset
const reset = () => {
  photo.value = null;
  result.value = null;
  cameraActive.value = false;
  isLoading.value = false;
  imageWidth.value = OPTIMAL_WIDTH;
  imageHeight.value = OPTIMAL_HEIGHT;
  [randomQuoteColor.value, randomCharacterColor.value] =
    getRandomDistinctColors();
};
</script>

<style>
.font-montserrat {
  font-family: "Montserrat", sans-serif;
}

/* from https://github.com/niklasvh/html2canvas/issues/2829 */
body > div:last-child > span + img {
  display: inline !important;
}
</style>
