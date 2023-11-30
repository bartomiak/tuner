<script setup>
import { ref, onUnmounted, computed } from 'vue';
import { PitchDetector } from 'pitchy';

    const currentPitch = ref(null);
    const isTuning = ref(false);
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

      let widthPercent = 50;

      if (frequencyDiff !== 0) {
        const diffPercent = (frequencyDiff / maxDiff) * 50;
        widthPercent += diffPercent;
        widthPercent = Math.max(0, Math.min(100, widthPercent));
      }

      console.log(Math.ceil(widthPercent))

      return { width: `${Math.ceil(widthPercent)}%` };
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
  <div>
    <button @click="startTuning" :disabled="isTuning" class="mr-2">Start Tuning</button>
    <button @click="stopTuning" :disabled="!isTuning">Stop Tuning</button>
    <div >Pitch: <span style="min-width: 200px;">{{ currentPitch }}</span> Hz</div>
    <div>{{ currentString }} string</div>

    <div style="background-color: grey; width: 100%; height: 20px; margin-top: 10px;">
      <div :style="tuningIndicatorStyle" style="height: 100%; background-color: blue;"></div>
    </div>
  </div>
</template>