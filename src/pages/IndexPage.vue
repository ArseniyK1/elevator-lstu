<template lang="pug">
q-page.row.text-black.window-height.window-width.bg-grey-5
  .col-4
    q-form.q-ma-md.bg-white.q-pa-md.rounded-borders.flex.column(@submit="createSession")
      div.flex.items-center
        q-icon(name="elevator" size="lg" color="primary" )
        .text-h4 Создание новой сессии лифта
      q-input(v-model='clientName', label='Ваш ник', stack-label='', :dense='dense'  )
      q-input(v-model='floors', label='Кол-во этажей', stack-label='', :dense='dense')
      q-input(v-model='timeFactor', label='Коэффициент замедления (по умолчанию - 1.0)', stack-label='', :dense='dense')
      q-btn.q-mt-sm(type="submit" color="primary") Создать
    .action
      q-btn(icon='arrow_upward', label="Наверх", @click="elevatorAction('up')", color="secondary" )
      q-btn(icon="arrow_downward", label="Вниз", @click="elevatorAction('down')", color="secondary" )
      q-btn(icon='block', label="Закрыть", @click="elevatorAction('close')", color="secondary" )
      q-btn(icon="meeting_room", label="Открыть", @click="elevatorAction('open')", color="secondary" )
      q-btn(icon="open_in_browser", label="Войти", color="secondary" )
      q-btn(icon="logout", label="Выйти", color="secondary" )
    div.state
      .text-h6 {{ stateElevator }}
  .col-8.text-h5.bg-grey-5 Текущее состояние лифта
    q-scroll-area(style="height: 90%; max-width: 100%;")
      q-card.my-card.q-mt-sm.q-mr-sm.q-ml-sm.shadow-14(style="min-height: 95%")
      q-card-section(horizontal='',)
        div.text-h6.scroll.flex.column.flex-center.wrap(style="width: 100%") Этаж
          div.text-h5.q-mb-lg.q-pa-md.rounded(v-for="item in countFloorsArr" v-if="session.sessionId" style="width: 25%; border: 1px solid black" )
            .text-h5(v-if="currentFloor" ) {{ currentFloor === item ? 'Лифт тут' : '' }}
            q-img.col-4(:src='currentImageElevator')
            q-btn.q-mt-md.q-ml-lg(icon="touch_app", color="secondary", @click="callElevatorFloor(item)" ) Вызвать на {{ item }} этаж
</template>

<script setup>
import { computed, onMounted, ref, watch } from "vue";
import { api } from "boot/axios";
import { useQuasar } from "quasar";

// Utils
const quasar = useQuasar();

// Form ref
const clientName = ref("");
const floors = ref("5");
const timeFactor = ref("");
const dense = ref(false);

// Elevator ref
const session = ref({});
const nextCycle = ref({
  MoveUpFast: false,
  MoveUpSlow: false,
  MoveDownFast: false,
  MoveDownSlow: false,
  DoClose: false,
  DoOpen: false
});
const arrayImageElevator = {
  elevator_up: "public/icons/elevator_up.svg",
  elevator_down: "public/icons/elevator_down.svg",
  elevator_entry_people: "public/icons/people-in-elevator.svg",
  elevator_open_no_people: "public/icons/elevator-open-no-peopls.svg",
  elevator_closed: "public/icons/elevator-closed.svg",
  elevator_alarm: "public/icons/elevator-alarm.svg"
};
const stateElevator = ref({
  sensors: {
    WeightSensor: 0,
    DoorOpened: false,
    DoorClosed: true,
    ObstacleSensor: false,
    FloorSensor: [false, false, false],
    ApproachSensor: [false, false, false],
    TopLimit: false,
    BottomLimit: false
  },
  emulation: {
    Position: 12.4,
    DriveDirection: "STOP",
    DriveDynamic: "STABLE",
    Door: "CLOSED",
    Alarm: null
  }
});
const currentImageElevator = ref("public/icons/elevator-closed.svg");

const countFloorsArr = ref([]);
const currentFloor = ref(0);

// ComputedValue

const sessionId = computed(() => session.value?.sessionId);
const elevatorSensors = computed(() => stateElevator.value.sensors);
const currentFloorsComputed = computed(() => currentFloor.value);

const createSession = async () => {
  const res = await api.post("/newSession", null, {
    params: {
      clientName: clientName.value || new Date(),
      floors: floors.value || "5",
      timeFactor: timeFactor.value || "1.0"
    }
  });
  session.value = res.data;

  // Создание массива с этажами, т.е если 5 этажей, то массив [1,2,3,4,5]
  if (+floors.value > 0) {
    for (let i = 1; +floors.value >= i; i++) {
      countFloorsArr.value.push(i);
    }
  } else {
    return [];
  }

  console.log(session.value.sessionId);
};

const nextCycleReq = async (selectedFloor = 0, action = "") => {
  const nextCycleRes = await api.put("/nextCycle", nextCycle.value, {
    params: { sessionId: session.value.sessionId }
  });

  stateElevator.value = nextCycleRes.data;

  const whileNotFoundFloor = stateElevator.value.sensors.ApproachSensor.some(
    el => el === true
  );

  if (!whileNotFoundFloor) {
    nextCycle.value.MoveDownFast = true;
  } else {
    nextCycle.value.MoveDownFast = false;
  }

  const initialFloors = stateElevator.value.sensors.ApproachSensor.map(
    (el, index) => {
      if (el === true) {
        return index;
      }
    }
  );

  for (let i = 0; i < initialFloors.length; i++) {
    if (!!initialFloors[i]) {
      currentFloor.value = initialFloors[i] + 1;
    }
  }
};

const elevatorAction = action => {
  switch (action) {
    case "up":
      currentImageElevator.value = arrayImageElevator.elevator_up;
      nextCycle.value.MoveUpFast = true;
      nextCycleReq();
      break;
    case "down":
      currentImageElevator.value = arrayImageElevator.elevator_down;
      nextCycle.value.MoveDownFast = true;
      nextCycleReq();
      break;
    case "close":
      currentImageElevator.value = arrayImageElevator.elevator_closed;
      nextCycle.value.DoClose = true;
      nextCycleReq();
      break;
    case "open":
      currentImageElevator.value = arrayImageElevator.elevator_open_no_people;
      nextCycle.value.DoOpen = true;
      nextCycleReq();
      break;
    default:
      currentImageElevator.value = arrayImageElevator.elevator_closed;
      break;
  }
};

const callElevatorFloor = async (floor = 0) => {
  if (+currentFloorsComputed.value > +floor) {
    if (!stateElevator.value.sensors.ApproachSensor[floor - 1]) {
      while (!elevatorSensors.value.ApproachSensor[floor - 1]) {
        if (!elevatorSensors.value.FloorSensor[floor - 1]) {
          nextCycle.value.MoveDownSlow = false;
          nextCycle.value.MoveDownFast = true;
          await nextCycleReq();
          console.log("вызов ф-ции");
        } else {
          nextCycle.value.MoveDownFast = false;
          nextCycle.value.MoveDownSlow = false;
          console.log("ПРИЕХАЛИ!");
        }
      }
    }
  } else if (+currentFloor.value === +floor) {
    console.log("Вы уже находитесь на этом этаже!");
  } else {
    if (!stateElevator.value.sensors.ApproachSensor[floor - 1]) {
      while (!elevatorSensors.value.ApproachSensor[floor - 1]) {
        if (!elevatorSensors.value.FloorSensor[floor - 1]) {
          nextCycle.value.MoveUpSlow = false;
          nextCycle.value.MoveUpFast = true;
          await nextCycleReq();
          console.log("вызов ф-ции");
        } else {
          nextCycle.value.MoveUpFast = false;
          nextCycle.value.MoveUpSlow = false;
          console.log("ПРИЕХАЛИ!");
        }
      }
    }
  }
};

const setIntervalNextCycle = setInterval(async () => {
  if (sessionId.value) {
    await nextCycleReq();
  }
}, 250);

watch(
  () => floors.value,
  (newValue, oldValue) => {
    if (newValue !== oldValue) {
      countFloorsArr.value = [];
      session.value = {};
      stateElevator.value = {};
    }
  }
);

// onMounted(async () => {
//   await createSession();
// });
</script>

<style>
* {
  box-sizing: border-box !important;
}
</style>
