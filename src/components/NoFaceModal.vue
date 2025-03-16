<template>
  <div
    v-if="isOpen"
    class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50"
  >
    <div class="bg-white p-6 rounded-lg shadow-lg text-center max-w-sm w-full">
      <h2 class="text-xl font-bold text-gray-800 mb-4">No Face Detected</h2>
      <p class="text-gray-600 mb-6">
        We couldn’t find a face in the photo. Would you like to try again?
      </p>
      <div class="flex justify-center gap-4">
        <button
          @click="retry"
          class="bg-blue-500 text-white px-4 py-2 rounded-lg shadow hover:bg-blue-600 transition"
        >
          Yes, Try Again
        </button>
        <button
          @click="cancel"
          class="bg-gray-500 text-white px-4 py-2 rounded-lg shadow hover:bg-gray-600 transition"
        >
          Proceed Anyway
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";

const isOpen = ref(false);

const emit = defineEmits(["retry", "cancel"]);

const open = () => {
  isOpen.value = true;
};

const retry = () => {
  isOpen.value = false;
  emit("retry");
};

const cancel = () => {
  isOpen.value = false;
  emit("cancel");
};

defineExpose({ open });
</script>

<style scoped>
/* Add any additional scoped styles if needed */
</style>
