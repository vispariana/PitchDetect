<script setup lang="ts">
import { ref, computed } from "vue";
const detune = ref(0);
const frq = ref(0);
const offseted = ref(0);

const centComp = computed(() => {
  return detune.value;
});

const noteComp = computed(() => {
  if (note.value === 0) return "";
  return (
    String(Math.floor(note.value / 12) - 1) + " " + noteStrings[note.value % 12]
  );
});

window.AudioContext = window.AudioContext || window.webkitAudioContext;

var audioContext = null;
var isPlaying = false;
var sourceNode = null;
var analyser = null;

var mediaStreamSource = null;
var detectorElem, pitchElem, noteElem, detuneElem, detuneAmount;

function startPitchDetect() {
  // grab an audio context
  audioContext = new AudioContext();

  // Attempt to get audio input
  navigator.mediaDevices
    .getUserMedia({
      audio: {
        mandatory: {
          googEchoCancellation: "false",
          googAutoGainControl: "false",
          googNoiseSuppression: "false",
          googHighpassFilter: "false",
        },
        optional: [],
      },
    })
    .then((stream) => {
      // Create an AudioNode from the stream.
      mediaStreamSource = audioContext.createMediaStreamSource(stream);

      // Connect it to the destination.
      analyser = audioContext.createAnalyser();
      analyser.fftSize = 2048;
      mediaStreamSource.connect(analyser);
      updatePitch();
    })
    .catch((err) => {
      // always check for errors at the end.
      console.error(`${err.name}: ${err.message}`);
      alert("Stream generation failed.");
    });
}

function toggleLiveInput() {
  if (isPlaying) {
    //stop playing and return
    sourceNode.stop(0);
    sourceNode = null;
    analyser = null;
    isPlaying = false;
    if (!window.cancelAnimationFrame)
      window.cancelAnimationFrame = window.webkitCancelAnimationFrame;
    window.cancelAnimationFrame(rafID);
  }
  getUserMedia(
    {
      audio: {
        mandatory: {
          googEchoCancellation: "false",
          googAutoGainControl: "false",
          googNoiseSuppression: "false",
          googHighpassFilter: "false",
        },
        optional: [],
      },
    },
    gotStream
  );
}

var rafID = null;
var tracks = null;
var buflen = 2048;
var buf = new Float32Array(buflen);

// var noteStrings = ["C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"];
var noteStrings = [
  "Re(♭)",
  "Re",
  "Mi(♭)",
  "Mi",
  "Fa",
  "Sol(♭)",
  "Sol",
  "La(♭)",
  "La",
  "Si(♭)",
  "Si",
  "Do",
];

function noteFromPitch(frequency) {
  var noteNum = 12 * (Math.log(frequency / 440) / Math.log(2));
  return Math.round(noteNum) + 69;
}

function frequencyFromNoteNumber(note) {
  return 440 * Math.pow(2, (note - 69) / 12);
}

function centsOffFromPitch(frequency, note) {
  return Math.floor(
    (1200 * Math.log(frequency / frequencyFromNoteNumber(note))) / Math.log(2)
  );
}

function autoCorrelate(buf, sampleRate) {
  // Implements the ACF2+ algorithm
  var SIZE = buf.length;
  var rms = 0;

  for (var i = 0; i < SIZE; i++) {
    var val = buf[i];
    rms += val * val;
  }
  rms = Math.sqrt(rms / SIZE);
  if (rms < 0.01)
    // not enough signal
    return -1;

  var r1 = 0,
    r2 = SIZE - 1,
    thres = 0.2;
  for (var i = 0; i < SIZE / 2; i++)
    if (Math.abs(buf[i]) < thres) {
      r1 = i;
      break;
    }
  for (var i = 1; i < SIZE / 2; i++)
    if (Math.abs(buf[SIZE - i]) < thres) {
      r2 = SIZE - i;
      break;
    }

  buf = buf.slice(r1, r2);
  SIZE = buf.length;

  var c = new Array(SIZE).fill(0);
  for (var i = 0; i < SIZE; i++)
    for (var j = 0; j < SIZE - i; j++) c[i] = c[i] + buf[j] * buf[j + i];

  var d = 0;
  while (c[d] > c[d + 1]) d++;
  var maxval = -1,
    maxpos = -1;
  for (var i = d; i < SIZE; i++) {
    if (c[i] > maxval) {
      maxval = c[i];
      maxpos = i;
    }
  }
  var T0 = maxpos;

  var x1 = c[T0 - 1],
    x2 = c[T0],
    x3 = c[T0 + 1];
  var a = (x1 + x3 - 2 * x2) / 2;
  var b = (x3 - x1) / 2;
  if (a) T0 = T0 - b / (2 * a);

  return sampleRate / T0;
}

const note = ref(0);

function updatePitch() {
  analyser.getFloatTimeDomainData(buf);
  var ac = autoCorrelate(buf, audioContext.sampleRate);

  if (ac == -1) {
  } else {
    let pitch = ac;
    frq.value = Math.round(pitch);
    note.value = noteFromPitch(pitch) - offseted.value;
    detune.value = centsOffFromPitch(pitch, note.value + offseted.value);
  }
  rafID = window.requestAnimationFrame(updatePitch);
}
</script>

<template>
  <div
    dir="rtl"
    style="
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 10px;
    "
  >
    <button
      style="width: 200px; height: 2rem; font-size: 1rem"
      @click="
        startPitchDetect();
        toggleLiveInput();
      "
    >
      شروع
    </button>
    <div>
      <label style="margin-left: 1rem">کوک</label>
      <select
        v-model="offseted"
        style="
          width: 150px;
          text-align: center;
          width: 200px;
          height: 2rem;
          font-size: 1rem;
        "
      >
        <option :value="1">دیاپازون</option>
        <option :value="0">نیم پرده بم</option>
        <option :value="-1">یک پرده بم</option>
      </select>
    </div>
    <div style="width: 100%; text-align: center">
      <div style="margin-top: 4rem; font-size: 1.25rem">فرکانس</div>
      <div style="font-size: 2rem">
        <span id="pitch" dir="ltr">{{ frq }} Hz</span>
      </div>
      <div style="margin-top: 4rem; font-size: 1.25rem">نت</div>
      <div style="font-size: 5rem; text-align: center" dir="ltr">
        {{ noteComp }}
      </div>
      <div style="margin-top: 4rem; font-size: 1.25rem">ناکوکی (سنت)</div>
      <div style="font-size: 3rem; text-align: center" dir="ltr">
        {{ centComp }}
      </div>
    </div>
  </div>
</template>

<style scoped></style>
