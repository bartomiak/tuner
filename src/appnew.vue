

<template>
  <div class="tuner">
    <!-- Dark Mode Toggle -->
    <label class="switch">
      <input type="checkbox" v-model="darkMode" @change="toggleDarkMode" />
      <span class="slider round"></span>
    </label>

    <!-- Tuner Display -->
    <div class="tuner-display">
      <div
        class="string-indicator"
        :class="{ 'too-low': isTooLow, 'in-tune': isInTune, 'too-high': isTooHigh }"
      ></div>
      <p>Current String: {{ currentString }}</p>
    </div>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue';
import ml5 from 'ml5'

export default {
  setup() {
    const model_url = 'https://cdn.jsdelivr.net/gh/ml5js/ml5-data-and-models/models/pitch-detection/crepe/';
    const darkMode = ref(false);
    const currentString = ref('E');
    const isTooLow = ref(false);
    const isInTune = ref(false);
    const isTooHigh = ref(false);
    let audioContext = null;

    const toggleDarkMode = () => {
      // Toggle dark mode styles
      darkMode.value = !darkMode.value;
    };

    const initializeAudioContext = async () => {
      try {
        // Initialize the audio context after user interaction (e.g., button click)
        await new Promise((resolve) => {
          const audioContextClass = window.AudioContext || window.webkitAudioContext;
          audioContext = new audioContextClass();
          resolve();
        });
      } catch (error) {
        console.error('Error initializing AudioContext:', error);
      }
    };

    const listenForPitch = async () => {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        const analyser = audioContext.createAnalyser();

        console.log(stream)
        console.log(analyser)
        const mic = audioContext.createMediaStreamSource(stream);
        mic.connect(analyser);
        analyser.connect(audioContext.destination);

        analyser.fftSize = 2048;
        const bufferLength = analyser.frequencyBinCount;
        const dataArray = new Uint8Array(bufferLength);

        const pitchDetectionModel = await ml5.pitchDetection(model_url, audioContext, mic);

        const updatePitch = () => {
          analyser.getByteTimeDomainData(dataArray);
          const pitch = pitchDetectionModel.getPitch(dataArray);

          if (pitch) {
            // Define pitch ranges for your strings
            const pitchRanges = {
              E: { min: 80, max: 100 },
              A: { min: 100, max: 120 },
              D: { min: 140, max: 160 },
              G: { min: 190, max: 210 },
              B: { min: 240, max: 260 },
              e: { min: 320, max: 340 },
            };

            for (const string in pitchRanges) {
              const { min, max } = pitchRanges[string];
              if (pitch >= min && pitch <= max) {
                currentString.value = string;
                break;
              }
            }

            const tolerance = 5;

            if (pitch > pitchRanges[currentString.value].max + tolerance) {
              isTooHigh.value = true;
              isInTune.value = false;
              isTooLow.value = false;
            } else if (pitch < pitchRanges[currentString.value].min - tolerance) {
              isTooLow.value = true;
              isInTune.value = false;
              isTooHigh.value = false;
            } else {
              isInTune.value = true;
              isTooLow.value = false;
              isTooHigh.value = false;
            }
          }

          requestAnimationFrame(updatePitch);
        };

        updatePitch();

      } catch (error) {
        console.error('Error accessing the microphone:', error);
      }
    };

    onMounted(async () => {
      await initializeAudioContext(); // Initialize the AudioContext first
      listenForPitch();
    });

    return {
      darkMode,
      currentString,
      isTooLow,
      isInTune,
      isTooHigh,
      toggleDarkMode,
    };
  },
};
</script>

<style scoped>
/* Tailwind CSS classes can be used here for styling */
</style>
