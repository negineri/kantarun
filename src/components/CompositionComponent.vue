<template>
  <div>
    <q-btn
      v-if="!isScanning"
      round
      icon="play_arrow"
      @click="startScanning"
    ></q-btn>
    <q-btn v-else round icon="pause" @click="stopScanning"></q-btn>

    <div class="q-mx-md ellipsis text-h2">Acceleration</div>
    <div class="q-mx-md">X: {{ acceleration?.x }}</div>
    <div class="q-mx-md">Y: {{ acceleration?.y }}</div>
    <div class="q-mx-md">Z: {{ acceleration?.z }}</div>
  </div>
</template>

<script lang="ts">
import { defineComponent, PropType, computed, ref, toRef, Ref } from 'vue';
import { Todo, Meta } from './models';

function useClickCount() {
  const clickCount = ref(0);
  function increment() {
    clickCount.value += 1;
    return clickCount.value;
  }

  return { clickCount, increment };
}

function useDisplayTodo(todos: Ref<Todo[]>) {
  const todoCount = computed(() => todos.value.length);
  return { todoCount };
}

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
  requestPermission: () => Promise<any>;
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
  setup(props) {
    const { acceleration, rotationRate, enableSensor, disableSensor } =
      useDeviceMotion();
    const isScanning = ref(false);

    const startScanning = () => {
      enableSensor();
      isScanning.value = true;
    };

    const stopScanning = () => {
      disableSensor();
      isScanning.value = false;
    };

    return {
      ...useClickCount(),
      ...useDisplayTodo(toRef(props, 'todos')),
      acceleration,
      rotationRate,
      stopScanning,
      startScanning,
      isScanning,
    };
  },
});
</script>
