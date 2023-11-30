<script setup>
import { ref, onUnmounted, computed } from 'vue';
import { PitchDetector } from 'pitchy';

const currentPitch = ref(null);
const isTuning = ref(false);
const feedback = ref('Come on!')
let audioContext = null;
let analyser = null;
let microphone = null;
let stream = null;
let animationFrameRequest = null;

const standardTuning = [82.41, 110, 146.83, 196, 246.94, 329.63];
const stringNames = ['E2', 'A2', 'D3', 'G3', 'B3', 'E4'];
const currentString = ref(null);

const getClosestStringName = (frequency) => {
  const closestNoteFrequency = getClosestNote(frequency);
  const index = standardTuning.findIndex(note => note === closestNoteFrequency);
  return stringNames[index];
};

const tuningIndicatorStyle = computed(() => {

  const currentFrequency = parseFloat(currentPitch.value);
  const closestNoteFrequency = getClosestNote(currentFrequency);
  const frequencyDiff = currentFrequency - closestNoteFrequency;
  const maxDiff = 10;
  let bgColor = 'green'

  let widthPercent = 50;

  if (frequencyDiff !== 0) {
    const diffPercent = (frequencyDiff / maxDiff) * 50;
    widthPercent += diffPercent;
    widthPercent = Math.max(0, Math.min(100, widthPercent));
  }

  if (widthPercent <= 45 || widthPercent >= 55) {
    bgColor = '#FD1444'
  } else if (widthPercent > 45 || widthPercent < 55) {
    bgColor = '#00E777'
  } else {
    bgColor = '#E7B90D'
  }

  return `left: ${Math.ceil(widthPercent)}%; background-color: ${bgColor}`;
});

const colorIndicator = computed(() => {

  const currentFrequency = parseFloat(currentPitch.value);
  const closestNoteFrequency = getClosestNote(currentFrequency);
  const frequencyDiff = currentFrequency - closestNoteFrequency;
  const maxDiff = 10;
  let color = 'white'

  let widthPercent = 50;

  if (frequencyDiff !== 0) {
    const diffPercent = (frequencyDiff / maxDiff) * 50;
    widthPercent += diffPercent;
    widthPercent = Math.max(0, Math.min(100, widthPercent));
  }

  if (widthPercent <= 45 || widthPercent >= 55) {
    color = '#FD1444'
  } else if (widthPercent > 45 || widthPercent < 55) {
    color = '#00E777'
  } else {
    color = '#E7B90D'
  }

  return `color: ${color}`;
});

const getClosestNote = (frequency) => {

  let closest = standardTuning[0];
  let smallestDiff = Math.abs(frequency - closest);

  for (let i = 1; i < standardTuning.length; i++) {
    let diff = Math.abs(frequency - standardTuning[i]);
    if (diff < smallestDiff) {
      smallestDiff = diff;
      closest = standardTuning[i];
    }
  }

  return closest;
};

const getTuningFeedback = (frequency, targetFrequency) => {
  const difference = frequency - targetFrequency;
  if (Math.abs(difference) < 1) {
    return "In Tune";
  } else if (difference > 0) {
    return "Too High";
  } else {
    return "Too Low";
  }
};

const startTuning = async () => {
  try {
    audioContext = new AudioContext();
    stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    microphone = audioContext.createMediaStreamSource(stream);
    analyser = audioContext.createAnalyser();
    analyser.fftSize = 2048;
    microphone.connect(analyser);
    const dataArray = new Float32Array(analyser.fftSize);

    const detectPitch = () => {
      if (!isTuning.value) {
        return;
      }

      analyser.getFloatTimeDomainData(dataArray);
      const detector = PitchDetector.forFloat32Array(analyser.fftSize, audioContext.sampleRate);
      const [pitch, clarity] = detector.findPitch(dataArray, audioContext.sampleRate);
      if (clarity > 0.97 && pitch) {
        currentString.value = getClosestStringName(pitch);
        const closestNote = getClosestNote(pitch);
        const tuningFeedback = getTuningFeedback(pitch, closestNote);
        console.log(tuningFeedback)
        feedback.value = tuningFeedback;
        currentPitch.value = `${pitch.toFixed(2)} Hz (${tuningFeedback})`;
      } else {
        currentPitch.value = null;
        currentString.value = null;
      }

      animationFrameRequest = requestAnimationFrame(detectPitch);
    };

    isTuning.value = true;
    detectPitch();
  } catch (error) {
    console.error('Error accessing microphone', error);
  }
};

const stopTuning = () => {
  if (animationFrameRequest) {
    cancelAnimationFrame(animationFrameRequest);
  }
  if (microphone && audioContext) {
    microphone.disconnect();
    audioContext.close();
  }

  if (stream) {
    stream.getTracks().forEach(track => track.stop());
  }

  isTuning.value = false;
  currentPitch.value = null;
};

onUnmounted(() => {
  stopTuning();
});
</script>

<template>
  <div class="flex flex-col items-center">
    <p class="h-12 text-white" :style="colorIndicator">
      {{ feedback }}
    </p>
    <div class="relative items-center flex justify-between max-w-3xl"
      style="width: 100%; height: 200px; margin-top: 10px;">
      <div :style="tuningIndicatorStyle" class="tuning-indicator transition-all h-full relative z-10"
        style="height: 100%;"></div>
      <div class="h-line absolute w-full bg-gray-800 z-0"></div>
      <div v-for="i in 30" class="scale-indicator bg-gray-800"></div>
    </div>
    <div class="flex justify-center">
      <div v-for="string in stringNames" :key="string" class="flex justify-center items-center mt-6">
        <div :class="currentString == string ? 'text-[#00E777] border-[#00E777]' : 'text-white'"
          class="mx-2 w-12 h-12 border rounded-full flex justify-center items-center">
          <p class="text-xl">{{ string }}</p>
        </div>
      </div>
    </div>
    <div class="mt-12">
      <button @click="startTuning" :disabled="isTuning"
        :class="!isTuning ? 'text-[#00E777] border-[#00E777]' : 'text-white'"
        class="border rounded-lg px-3 py-2 mr-2">Start Tuning</button>
      <button @click="stopTuning" :disabled="!isTuning" :class="isTuning ? 'text-red-500 border-red-400' : 'text-white'"
        class="border rounded-lg px-3 py-2">Stop Tuning</button>
    </div>
  </div>
</template>

<style>
.h-line {
  height: 1px;
  bottom: 50%;
}

.tuning-indicator {
  width: 1px;
}

.scale-indicator {
  width: 1px;
  height: 40%;
}

.scale-indicator:nth-child(odd) {
  width: 1px;
  height: 60%;
}</style>