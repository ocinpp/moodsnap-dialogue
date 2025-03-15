<template>
  <div
    class="min-h-screen bg-gray-100 flex flex-col items-center justify-center p-4 font-montserrat"
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
      <h1 class="text-4xl font-bold mb-4 text-gray-800">
        MoodSnap Movie Quote Generator
      </h1>
      <p class="mb-6 text-gray-600">Capture your mood and get a movie quote!</p>
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
        class="relative rounded-lg shadow-lg overflow-hidden"
        :style="{
          width: `${imageWidth}px`,
          height: `${imageHeight}px`,
        }"
      >
        <img
          v-if="photo"
          :src="photo"
          class="w-full h-full object-cover"
          alt="Result"
        />
        <div class="absolute inset-0 flex flex-col justify-end text-white p-4">
          <p
            class="font-bold text-lg leading-tight px-2 py-1"
            :style="{ backgroundColor: randomOverlayColor }"
          >
            "{{ result.quote }}"
          </p>
          <p
            v-if="result.translation"
            class="text-sm italic leading-tight px-2 py-2"
            :style="{ backgroundColor: randomOverlayColor }"
          >
            {{ result.translation }}
          </p>
          <p
            class="text-sm leading-tight px-2 py-2"
            :style="{ backgroundColor: randomBackgroundColor }"
          >
            {{ result.movie }} - {{ result.character }}
          </p>
        </div>
      </div>
      <p class="mt-4 text-gray-700">
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
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import html2canvas from "html2canvas";
import * as faceapi from "@vladmandic/face-api";

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

// Random background and overlay colors
const colors = ["#145DA0", "#0c2d48", "#2e8bc0", "#b1d4e0"];
const randomBackgroundColor = `${
  colors[Math.floor(Math.random() * colors.length)]
}CC`; // 90% opacity
const randomOverlayColor = `${
  colors[Math.floor(Math.random() * colors.length)]
}E6`; // 90% opacity

// Update screenWidth on resize
onMounted(() => {
  window.addEventListener("resize", () => {
    screenWidth.value = window.innerWidth - 32;
  });
});

// Merged Quote Database (English + Cantonese)
const quotes = {
  happy: [
    {
      quote: "Here's looking at you, kid.",
      movie: "Casablanca",
      character: "Rick Blaine",
    },
    {
      quote: "May the Force be with you.",
      movie: "Star Wars",
      character: "Han Solo",
    },
    {
      quote: "I'm on top of the world!",
      movie: "White Heat",
      character: "Cody Jarrett",
    },
    {
      quote: "做人如果無夢想，同條鹹魚有咩分別啊？",
      romanization:
        "Zou6 jan4 jyu4 gwo2 mou5 mung6 soeng2, tung4 tiu4 haam4 jyu2 jau5 me1 fan1 bit6 aa3?",
      translation:
        "If a person has no dreams, how are they different from a salted fish?",
      movie: "Shaolin Soccer",
      character: "Sing",
    },
    {
      quote: "我同你講，人生最緊要係開心！",
      romanization:
        "Ngo5 tung4 nei5 gong2, jan4 sang1 zeoi3 gan2 jiu3 hai6 hoi1 sam1!",
      translation:
        "I’m telling you, the most important thing in life is to be happy!",
      movie: "The God of Cookery",
      character: "Stephen Chow",
    },
    {
      quote: "有信心就一定會贏！",
      romanization: "Jau5 seon3 sam1 zau6 jat1 ding6 wui5 jeng4!",
      translation: "With confidence, you’ll definitely win!",
      movie: "Kung Fu Hustle",
      character: "Sing",
    },
    {
      quote: "今日我好開心，因為我終於做到我想做嘅嘢！",
      romanization:
        "Gam1 jat6 ngo5 hou2 hoi1 sam1, jan1 wai6 ngo5 zung1 jyu1 zou6 dou3 ngo5 soeng2 zou6 ge3 je5!",
      translation:
        "Today I’m so happy because I finally did what I wanted to do!",
      movie: "Fight Back to School",
      character: "Star Chow",
    },
    {
      quote: "笑一笑，世界更美妙！",
      romanization: "Siu3 jat1 siu3, sai3 gaai3 gang3 mei5 miu6!",
      translation: "Smile a little, and the world becomes more wonderful!",
      movie: "Flirting Scholar",
      character: "Tong Pak Fu",
    },
  ],
  sad: [
    {
      quote: "I'm king of the world!",
      movie: "Titanic",
      character: "Jack Dawson",
    },
    {
      quote: "I'll never let go, Jack.",
      movie: "Titanic",
      character: "Rose DeWitt",
    },
    {
      quote: "Why so serious?",
      movie: "The Dark Knight",
      character: "The Joker",
    },
    {
      quote: "那個時代已過去。屬於那個時代的一切，都不存在了。",
      romanization:
        "Naa5 go3 si4 doi6 ji5 gwo3 heoi3. Suk6 jyu1 naa5 go3 si4 doi6 dik1 jat1 cai3, dou1 bat1 cyun4 zoi6 liu5.",
      translation:
        "That era is gone. Everything that belongs to that era no longer exists.",
      movie: "In the Mood for Love",
      character: "Narrator",
    },
    {
      quote: "人生有幾多個十年？",
      romanization: "Jan4 sang1 jau5 gei2 do1 go3 sap6 nin4?",
      translation: "How many decades are there in a lifetime?",
      movie: "Days of Being Wild",
      character: "Yuddy",
    },
    {
      quote: "我會唔會再見到你？",
      romanization: "Ngo5 wui5 m4 wui5 zoi3 gin3 dou2 nei5?",
      translation: "Will I ever see you again?",
      movie: "Chungking Express",
      character: "Cop 663",
    },
    {
      quote: "有啲嘢，失去了就唔會再返嚟。",
      romanization:
        "Jau5 di1 je5, sat1 heoi3 liu5 zau6 m4 wui5 zoi3 faan1 lei4.",
      translation: "Some things, once lost, will never come back.",
      movie: "Comrades: Almost a Love Story",
      character: "Li Xiao-jun",
    },
    {
      quote: "點解每次我想要嘅嘢，都要失去？",
      romanization:
        "Dim2 gaai2 mui5 ci3 ngo5 soeng2 jiu3 ge3 je5, dou1 jiu3 sat1 heoi3?",
      translation: "Why do I have to lose everything I want every time?",
      movie: "Fallen Angels",
      character: "Wong Chi-ming",
    },
  ],
  angry: [
    {
      quote: "You talking to me?",
      movie: "Taxi Driver",
      character: "Travis Bickle",
    },
    {
      quote: "Say hello to my little friend!",
      movie: "Scarface",
      character: "Tony Montana",
    },
    {
      quote: "出嚟行，遲早要還！",
      romanization: "Ceot1 lei4 haang4, ci4 zou2 jiu3 waan4!",
      translation:
        "If you step into this world, sooner or later you’ll have to pay it back!",
      movie: "Infernal Affairs",
      character: "Hon Sam",
    },
    {
      quote: "你同我講，邊個夠膽同我爭！",
      romanization:
        "Nei5 tung4 nei5 gong2, bin1 go3 gau3 daam2 tung4 ngo5 zaang1!",
      translation: "Tell me, who dares to fight me!",
      movie: "Election",
      character: "Lok",
    },
    {
      quote: "我同你講，唔好搞亂我！",
      romanization: "Ngo5 tung4 nei5 gong2, m4 hou2 gaau2 lyun6 ngo5!",
      translation: "I’m telling you, don’t mess with me!",
      movie: "Police Story",
      character: "Chan Ka-Kui",
    },
    {
      quote: "你再唔走，我同你唔客氣！",
      romanization: "Nei5 zoi3 m4 zau2, ngo5 tung4 nei5 m4 haak3 hei3!",
      translation: "If you don’t leave now, I won’t be polite with you!",
      movie: "Drunken Master",
      character: "Wong Fei-hung",
    },
    {
      quote: "我唔係怕你，我係唔想同你嘈！",
      romanization:
        "Ngo5 m4 hai6 paa3 nei5, ngo5 hai6 m4 soeng2 tung4 nei5 cou4!",
      translation:
        "I’m not afraid of you, I just don’t want to argue with you!",
      movie: "Young and Dangerous",
      character: "Chan Ho-nam",
    },
  ],
  surprised: [
    {
      quote: "I see dead people.",
      movie: "The Sixth Sense",
      character: "Cole Sear",
    },
    {
      quote: "You're gonna need a bigger boat.",
      movie: "Jaws",
      character: "Chief Brody",
    },
    {
      quote: "點解會係咁樣？",
      romanization: "Dim2 gaai2 wui5 hai6 gam2 joeng2?",
      translation: "Why is it like this?",
      movie: "A Simple Life",
      character: "Roger",
    },
    {
      quote: "你話咩？真係唔敢信！",
      romanization: "Nei5 waa6 me1? Zan1 hai6 m4 gam2 seon3!",
      translation: "What did you say? I can’t believe it!",
      movie: "The Killer",
      character: "Ah Jong",
    },
    {
      quote: "吓？原来係咁！",
      romanization: "Haa2? Jyun4 loi4 hai6 gam2!",
      translation: "Huh? So that’s how it is!",
      movie: "Hard Boiled",
      character: "Tequila",
    },
    {
      quote: "乜嘢？真係你做嘅？",
      romanization: "Mat1 je5? Zan1 hai6 nei5 zou6 ge3?",
      translation: "What? You really did it?",
      movie: "Project A",
      character: "Dragon Ma",
    },
    {
      quote: "吓！點解會有咁嘅事？",
      romanization: "Haa2! Dim2 gaai2 wui5 jau5 gam2 ge3 si6?",
      translation: "What! How could something like this happen?",
      movie: "Ashes of Time",
      character: "Ouyang Feng",
    },
  ],
  neutral: [
    {
      quote: "Houston, we have a problem.",
      movie: "Apollo 13",
      character: "Jim Lovell",
    },
    { quote: "Just keep swimming.", movie: "Finding Nemo", character: "Dory" },
    {
      quote: "講你又唔聽，聽你又唔懂，懂你又唔做。",
      romanization:
        "Gong2 nei5 jau6 m4 teng1, teng1 nei5 jau6 m4 dung2, dung2 nei5 jau6 m4 zou6.",
      translation:
        "I talk but you don’t listen. You listen but don’t understand. You understand but don’t act.",
      movie: "The Bare-Footed Kid",
      character: "Unknown",
    },
    {
      quote: "人生如戲，戲如人生。",
      romanization: "Jan4 sang1 jyu4 hei3, hei3 jyu4 jan4 sang1.",
      translation: "Life is like a play, and a play is like life.",
      movie: "Echoes of the Rainbow",
      character: "Narrator",
    },
    {
      quote: "有啲嘢，唔使問咁多。",
      romanization: "Jau5 di1 je5, m4 sai2 man6 gam3 do1.",
      translation: "Some things don’t need so many questions.",
      movie: "Exiled",
      character: "Blaze",
    },
    {
      quote: "人生就係一場遊戲，點玩都得。",
      romanization:
        "Jan4 sang1 zau6 hai6 jat1 coeng4 jau4 hei3, dim2 waan2 dou1 dak1.",
      translation: "Life is just a game, you can play it however you want.",
      movie: "The Longest Nite",
      character: "Sam",
    },
    {
      quote: "做人要低調，先至唔會有咁多麻煩。",
      romanization:
        "Zou6 jan4 jiu3 dai1 diu6, sin1 zi3 m4 wui5 jau5 gam3 do1 maa4 faan4.",
      translation:
        "To live, keep a low profile, then you won’t have so much trouble.",
      movie: "Running Out of Time",
      character: "Cheung",
    },
  ],
};

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

// Load Face-API Models with Loading State
const loadModels = async () => {
  isModelLoading.value = true;
  await faceapi.nets.ssdMobilenetv1.loadFromUri("/models");
  await faceapi.nets.faceExpressionNet.loadFromUri("/models");
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

// Mood Analysis with Face-API
const analyzeMood = async () => {
  isLoading.value = true;
  await loadModels();
  const img = new Image();
  img.src = photo.value;
  img.onload = async () => {
    console.log("Image loaded for analysis:", img.width, img.height);
    const detections = await faceapi
      .detectSingleFace(img)
      .withFaceExpressions();
    if (detections) {
      const expressions = detections.expressions;
      const mood = Object.entries(expressions).reduce((a, b) =>
        a[1] > b[1] ? a : b
      )[0];
      const moodQuotes = quotes[mood] || quotes.neutral;
      const randomQuote =
        moodQuotes[Math.floor(Math.random() * moodQuotes.length)];
      result.value = { mood, ...randomQuote };
    } else {
      result.value = {
        mood: "neutral",
        ...quotes.neutral[Math.floor(Math.random() * quotes.neutral.length)],
      };
    }
    isLoading.value = false;
  };
  img.onerror = () => {
    console.error("Failed to load image for analysis");
    isLoading.value = false;
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
};
</script>

<style>
.font-montserrat {
  font-family: "Montserrat", sans-serif;
}
body > div:last-child > span + img {
  display: inline !important;
}
</style>
