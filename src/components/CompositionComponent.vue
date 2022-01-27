<template>
  <div>
    <q-btn
      v-if="!isScanning"
      round
      icon="play_arrow"
      @click="startScanning"
    ></q-btn>
    <q-btn v-else round icon="pause" @click="stopScanning"></q-btn>
    <q-icon
      v-if="isStepping"
      name="directions_walk"
      class="text-primary"
      size="xl"
    ></q-icon>
    <q-icon v-else name="directions_walk" size="xl"></q-icon>
    <q-badge color="secondary"> Model: {{ threshold }} (0 to 50) </q-badge>

    <q-slider v-model="threshold" :min="0" :max="50" />

    <div class="q-mx-md">step count: {{ steptimeHistory.length }}</div>
    <div class="q-mx-md">BPM: {{ runningBPM }}</div>
    <div class="q-mx-md">Title: {{ songData?.song.name }}</div>
    <div class="q-mx-md">
      Link: <a v-bind:href="songData?.song.url">{{ songData?.song.url }}</a>
    </div>
    <div class="q-mx-md ellipsis text-h4">Acceleration</div>
    <div class="q-mx-md">X: {{ acceleration?.x }}</div>
    <div class="q-mx-md">Y: {{ acceleration?.y }}</div>
    <div class="q-mx-md">Z: {{ acceleration?.z }}</div>
  </div>
</template>

<script lang="ts">
import { defineComponent, PropType, computed, ref, watch } from 'vue';
import { Todo, Meta } from './models';
import axios, { AxiosResponse } from 'axios';

interface DeviceMotionOptions {
  throttleMs: number;
}

type DeviceMotionEventAccelerationRW = {
  -readonly [P in keyof DeviceMotionEventAcceleration]: DeviceMotionEventAcceleration[P];
};

type DeviceMotionEventRotationRateRW = {
  -readonly [P in keyof DeviceMotionEventRotationRate]: DeviceMotionEventRotationRate[P];
};

interface DeviceMotionEventIos extends DeviceMotionEvent {
  requestPermission(): Promise<PermissionState>;
}

function useDeviceMotion(options?: DeviceMotionOptions) {
  if (options === void 0) {
    options = { throttleMs: 10 };
  }
  const acceleration = ref<DeviceMotionEventAccelerationRW>();
  const rotationRate = ref<DeviceMotionEventRotationRateRW>();
  const interval = ref(0);
  const accelerationIncludingGravity = ref<DeviceMotionEventAccelerationRW>();
  function onDeviceMotion(event: DeviceMotionEvent) {
    acceleration.value = event.acceleration as DeviceMotionEventAccelerationRW;
    accelerationIncludingGravity.value =
      event.accelerationIncludingGravity as DeviceMotionEventAccelerationRW;
    rotationRate.value = event.rotationRate as DeviceMotionEventRotationRateRW;
    interval.value = event.interval;
  }
  let handler = onDeviceMotion;
  const enableSensor = function () {
    if (DeviceMotionEvent && 'requestPermission' in DeviceMotionEvent) {
      let DeviceMotionEventIos =
        DeviceMotionEvent as unknown as DeviceMotionEventIos;
      DeviceMotionEventIos.requestPermission()
        .then((permissionState) => {
          if (permissionState === 'granted') {
            window.addEventListener('devicemotion', handler, false);
          } else {
            console.log('Perrmission not granted!');
            alert('Perrmission not granted!');
          }
        })
        .catch(console.error);
    } else {
      //For other devices
      console.log('detected other device. so adding listener...');
      window.addEventListener('devicemotion', handler, false);
    }
  };

  const disableSensor = function () {
    window.removeEventListener('devicemotion', handler, false);
  };

  // onMounted();
  // onUnmounted();
  return {
    acceleration: acceleration,
    accelerationIncludingGravity: accelerationIncludingGravity,
    rotationRate: rotationRate,
    interval: interval,
    enableSensor: enableSensor,
    disableSensor: disableSensor,
  };
}

interface songJson {
  song: {
    name: string;
    tempo: string;
    url: string;
  };
}

export default defineComponent({
  name: 'CompositionComponent',
  props: {
    title: {
      type: String,
      required: true,
    },
    todos: {
      type: Array as PropType<Todo[]>,
      default: () => [],
    },
    meta: {
      type: Object as PropType<Meta>,
      required: true,
    },
    active: {
      type: Boolean,
    },
  },
  setup() {
    const { acceleration, enableSensor, disableSensor } = useDeviceMotion();
    const isScanning = ref(false);
    const threshold = ref(25);
    const isStepping = computed(() => {
      const ax = acceleration.value?.x ? acceleration.value.x : 0;
      const ay = acceleration.value?.y ? acceleration.value.y : 0;
      const az = acceleration.value?.z ? acceleration.value.z : 0;
      if (Math.abs(ax) + Math.abs(ay) + Math.abs(az) > threshold.value) {
        return true;
      } else {
        return false;
      }
    });
    const steptimeHistory = ref<Date[]>([]);
    const runningBPM = ref(0);
    const songData = ref<songJson>();

    watch(isStepping, () => {
      if (isStepping.value) {
        let currentTime = new Date();
        if (
          currentTime.getTime() -
            steptimeHistory.value[steptimeHistory.value.length - 1].getTime() >
          250
        ) {
          steptimeHistory.value.push(currentTime);
        }
      }
      if (steptimeHistory.value.length > 10) {
        let shMin = 9999999999999;
        let shMax = 0;
        for (
          let index = steptimeHistory.value.length - 6;
          index < steptimeHistory.value.length;
          index++
        ) {
          let shTemp =
            steptimeHistory.value[index].getTime() -
            steptimeHistory.value[index - 1].getTime();
          if (shTemp < shMin) shMin = shTemp;
          if (shTemp > shMax) shMax = shTemp;
        }
        if (shMax - shMin < 100) {
          runningBPM.value = Math.round(60000 / ((shMax + shMin) / 2));
          stopScanning();
          getMusic(runningBPM.value)
            .then((res) => {
              songData.value = res;
            })
            .catch((e) => {
              console.error(e);
            });
        }
      }
    });

    const getMusic = async (bpm: number) => {
      let songJ: AxiosResponse<songJson>;
      try {
        [songJ] = await Promise.all([
          axios.get<songJson>(`${process.env.VUE_APP_JSON_SERVER}/song`, {
            params: {
              bpm: bpm,
            },
          }),
        ]);
      } catch (err) {
        throw err;
      }
      return songJ.data;
    };

    const startScanning = () => {
      enableSensor();
      steptimeHistory.value = [new Date()];
      isScanning.value = true;
    };

    const stopScanning = () => {
      disableSensor();
      isScanning.value = false;
    };

    return {
      acceleration,
      stopScanning,
      startScanning,
      isScanning,
      isStepping,
      threshold,
      steptimeHistory,
      runningBPM,
      songData,
    };
  },
});
</script>
