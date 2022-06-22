<template>
  <div ref="wrapper" class="wrapper" :class="{
    'playing': isPlaying
  }">
    <canvas ref="vizplayer" class="vizplayer" />
    <img
        v-show="coverImg"
        :src="coverImg"
        alt="Audio Cover"
        ref="cover"
        class="cover"
        crossorigin="anonymous"
        @load="getCoverColor"
    />
    <div class="control-bar">
      <div class="button" @click="togglePlay">
        <i v-if="isPlaying" class="bx bx-pause" />
        <i v-else class="bx bx-play" />
      </div>
      <div class="time">
        <span ref="timeNow" class="now">00:00</span>
        <span class="divider">/</span>
        <span ref="timeFull" class="full">00:00</span>
      </div>
      <div
          class="process-bar"
          @mouseenter="startSeeking"
          @mouseleave="endSeeking"
          @mousemove="seekProgress"
          @click="jumpProgress"
      >
        <div ref="barFull" class="bar full" />
        <div ref="barBuffered" class="bar buffered" />
        <div ref="barPlayed" class="bar played" />
        <div ref="seeker" class="seeker">
          <div class="ball" />
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue';
import Stats from 'stats.js';
import ColorThief from "colorthief";
import {hslToRgb, rgbToHsl} from "../utils/color";

const props = defineProps({
  audioSrc: String,
  cover: String,
});

const coverImg = ref(props.cover);

const wrapper = ref<HTMLDivElement | null>(null);
const vizplayer = ref<HTMLCanvasElement | null>(null);
const cover = ref<HTMLImageElement | null>(null);

const timeNow = ref<HTMLSpanElement | null>(null);
const timeFull = ref<HTMLSpanElement | null>(null);
const barFull = ref<HTMLDivElement | null>(null);
const barBuffered = ref<HTMLDivElement | null>(null);
const barPlayed = ref<HTMLDivElement | null>(null);
const seeker = ref<HTMLDivElement | null>(null);

onMounted(() => {
  if (props.audioSrc) {
    // Initialize
    console.log('Viz mounted');
    init(props.audioSrc);
  } else {
    console.error('audio is required');
  }
});

// Define constants
const FREQ_BIN_COUNT = 256; // Half of FFT size
const COLORS = {
  bg: '#000c',
  freq: '#fff',
  fragments: '#fff3',
};
const ANGLE_STEP = 0.3; // Change angle per 20ms
const RADIUS_LIMIT = {
  min: 1/5,
  max: 1/3,
};

// Define variables
let analyser: AnalyserNode;
let audio: HTMLAudioElement;
let context: AudioContext;
let cctx: CanvasRenderingContext2D;
let canvasSize: {
  width: number,
  height: number,
};
let freqMultiplyRate: {
  radius: number,
  angleInDegrees: number,
};

// Define state
let isContextResumed = false;
let angleOffset = 0;
let angleStepEvent: number = 0;
const isPlaying = ref(false);
let isSeeking = false;

// Define performance monitor
let stats: Stats;

// Prepare data array
const dataArray = new Uint8Array(FREQ_BIN_COUNT);

// Initialize function
const init = (src: string) => {
  initCanvasSize(wrapper.value!.clientWidth);
  initAudioAnalyser(src);

  window.addEventListener('resize', () => {
    if (wrapper.value?.clientWidth) {
      initCanvasSize(wrapper.value.clientWidth);
    }
  });

  initStats();

  // Start render
  requestAnimationFrame(render);
};

const initCanvasSize = (width: number) => {
  canvasSize = {
    width,
    height: width / 16 * 9, // Reserve space for status bar
  };
  vizplayer.value!.width = canvasSize.width;
  vizplayer.value!.height = canvasSize.height;
  freqMultiplyRate = {
    radius: canvasSize.height * (RADIUS_LIMIT.max - RADIUS_LIMIT.min) / 256, // (2/3-1/4)/2=5/24 , 0 - 255
    angleInDegrees: 360 / FREQ_BIN_COUNT,
  }
  cctx = vizplayer.value!.getContext('2d')!;

  // Set cover size
  cover.value!.width = canvasSize.height * RADIUS_LIMIT.min * 2 - 4;
  cover.value!.height = canvasSize.height * RADIUS_LIMIT.min * 2 - 4;

  // Set line style
  cctx.strokeStyle = COLORS.freq;
  cctx.lineWidth = canvasSize.height / 4 * 2 * Math.PI / FREQ_BIN_COUNT * 2 / 3;
  cctx.lineJoin = 'round';
  cctx.lineCap = 'round';
};

const initAudioAnalyser = (src: string) => {
  // Load media
  audio = new Audio(src);
  audio.autoplay = false;

  // Cross browser support
  // @ts-ignore
  const AudioContext = window.AudioContext || window.webkitAudioContext;

  // Init Audio Context
  context = new AudioContext();

  // Create element
  const source = context.createMediaElementSource(audio);
  analyser = context.createAnalyser();

  // Connect chain
  source.connect(analyser);
  analyser.connect(context.destination);

  // Initialize analyser
  analyser.fftSize = FREQ_BIN_COUNT << 1;

  // Update status when audio status changed
  audio.addEventListener('pause', () => {
    isPlaying.value = false;
    stopAngleStep();
  });
  audio.addEventListener('play', () => {
    isPlaying.value = true;
    startAngleStep();
  });
}

const initStats = () => {
  stats = new Stats();
  stats.showPanel(0); // 0: fps, 1: ms, 2: mb, 3+: custom
  document.body.appendChild(stats.dom);
}

const startAngleStep = () => {
  if (angleStepEvent === 0) {
    angleStepEvent = setInterval(() => {
      angleOffset += ANGLE_STEP;
    }, 20);
  }
}

const stopAngleStep = () => {
  if (angleStepEvent) {
    clearInterval(angleStepEvent);
    angleStepEvent = 0;
  }
}

const render = () => {
  stats.begin();
  // Get Frequency data
  analyser.getByteFrequencyData(dataArray);

  // Clear canvas
  cctx.clearRect(0, 0, canvasSize.width, canvasSize.height);
  cctx.fillStyle = COLORS.bg;
  cctx.fillRect(0, 0, canvasSize.width, canvasSize.height);

  // Draw frequency data
  cctx.fillStyle = COLORS.freq;
  for (let i = 0; i < FREQ_BIN_COUNT / 2; i++) {
    // Only select lower half to optimize visual effects
    // (otherwise it will be too empty)
    cctx.beginPath();
    const angleInDegree = freqMultiplyRate.angleInDegrees * i * 2 + angleOffset;
    const startPoint = convertPolarToCartesian(
      canvasSize.height * RADIUS_LIMIT.min,
      angleInDegree,
        canvasSize.width / 2,
        canvasSize.height / 2,
    );
    cctx.moveTo(startPoint.x, startPoint.y); // Start point
    const endPoint = convertPolarToCartesian(
        dataArray[i] * freqMultiplyRate.radius + canvasSize.height * RADIUS_LIMIT.min,
        angleInDegree,
        canvasSize.width / 2,
        canvasSize.height / 2,
    );
    cctx.lineTo(
        endPoint.x, endPoint.y,
    );
    cctx.stroke();
    cctx.closePath();
  }

  // Update progress
  //// Update time
  timeFull.value!.innerText = parseSecondsToTime(audio.duration);
  //// Update played progress
  const playedPercent = audio.currentTime / audio.duration * 100;
  barPlayed.value!.style.width = `${playedPercent}%`;
  if (!isSeeking) {
    timeNow.value!.innerText = parseSecondsToTime(audio.currentTime);
    seeker.value!.style.left = `${playedPercent}%`;
  }
  //// Update buffered progress
  ////// Find latest buffer
  let latestBuffered = 0;
  for (let i = 0; i < audio.buffered.length; i++) {
    if (audio.buffered.end(i) > latestBuffered) {
      latestBuffered = audio.buffered.end(i);
    }
  }
  ////// Update progress style
  barBuffered.value!.style.width = `${latestBuffered / audio.duration * 100}%`;

  stats.end();

  requestAnimationFrame(render);
}

const convertPolarToCartesian = (radius: number, angleInDegrees: number, offsetX: number, offsetY: number) => {
  const angleInRadians = (angleInDegrees - 90) * (Math.PI / 180);

  return {
    x: radius * Math.cos(angleInRadians) + offsetX,
    y: radius * Math.sin(angleInRadians) + offsetY,
  };
};

const parseSecondsToTime = (seconds: number): string => {
  const sec = Math.floor(seconds % 60);
  const min = Math.floor(seconds / 60);
  return `${min < 10 ? '0' : ''}${min}:${sec < 10 ? '0' : ''}${sec}`;
};

const togglePlay = () => {
  if (audio.paused) {
    if (!isContextResumed) {
      // Resume audio
      context.resume();
      // Update status
      isContextResumed = true;
    }

    audio.play();
  } else {
    audio.pause();
  }
};

const startSeeking = () => {
  isSeeking = true;
}

const endSeeking = () => {
  isSeeking = false;
}

const seekProgress = (e: MouseEvent) => {
  const seekPercent = e.offsetX / barFull.value!.clientWidth;
  const seekTime = Math.floor(seekPercent * audio.duration);
  timeNow.value!.innerText = parseSecondsToTime(seekTime);
  barPlayed.value!.style.width = `${seekPercent * 100}%`;
  seeker.value!.style.left = `${seekPercent * 100}%`;
};

const jumpProgress = (e: MouseEvent) => {
  const seekPercent = e.offsetX / barFull.value!.clientWidth;
  audio.currentTime = seekPercent * audio.duration;
};

const getCoverColor = () => {
  const colorThief = new ColorThief();
  const colors = colorThief.getPalette(cover.value);
  COLORS.bg = `rgb(${changeColor(colors[0], true).join(',')})`;
  COLORS.freq = `rgb(${changeColor(colors[1], false).join(',')})`;
  COLORS.fragments = `rgba(${changeColor(colors[2], false).join(',')},0.3)`;
}

const changeColor = (rgbColor: [number, number, number], isDarken: boolean): [number, number, number] => {
  const [h, s] = rgbToHsl(...rgbColor);
  return hslToRgb(h, s, isDarken ? 0.25 : 0.75);
}

</script>
<style scoped lang="less">
.wrapper {
  overflow: hidden;
  position: relative;
}
.vizplayer {
  display: block;
}
.cover {
  position: absolute;
  border-radius: 100%;
  top: 0;
  bottom: 0;
  left: 0;
  right: 0;
  margin: auto;
  object-fit: cover;
}
.control-bar {
  width: 100%;
  position: relative;

  > * {
    position: absolute;
  }
  > .process-bar {
    position: relative;
    width: 100%;
    cursor: pointer;
    > * {
      position: absolute;
    }
    > .bar {
      bottom: 0;
      height: .15rem;
      transition: height .3s ease-in-out;

      &.full {
        width: 100%;
        background-color: #888;
      }
      &.buffered {
        background-color: #bbb;
        pointer-events: none;
      }
      &.played {
        background-color: #ddd;
        pointer-events: none;
      }
    }
    > .seeker {
      bottom: 0;
      transition: bottom .3s ease-in-out;
      pointer-events: none;
      > .ball {
        position: absolute;
        width: .4rem;
        height: .4rem;
        border-radius: 100%;
        top: -.4rem;
        left: -.2rem;
        background-color: #fff;
        border: 1px solid #3333;
      }
    }
  }
  .time {
    opacity: 0;
    transition: opacity .3s;
    color: #fffc;
    right: 0.5rem;
    top: -2rem;
    user-select: none;

    > span {
      margin: 2px;
    }

    > .divider {
      font-weight: bold;
    }
  }
  .button {
    cursor: pointer;
    width: 1.8rem;
    height: 1.8rem;
    display: flex;
    justify-content: center;
    align-items: center;
    background-color: #aaa6;
    border-radius: 100%;
    font-size: 1.5rem;
    opacity: 0;
    transition: opacity .3s ease-in-out;
    color: #fffc;
    left: 0.6rem;
    top: -2.7rem;
  }
}

.wrapper:not(.playing), .wrapper.playing:hover {
  .control-bar {
    > .process-bar {
      > .bar {
        height: .3rem;
      }
      > .seeker {
        bottom: .15rem;
      }
    }
    .time, .button {
      opacity: 1;
    }
  }
}
</style>
