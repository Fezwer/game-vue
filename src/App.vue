<template>
  <div class="page">
    <h1 class="title">ЗМЕЙКА</h1>
    <div v-if="!hasEnteredName" class="name-input">
      <input type="text" v-model="playerName" placeholder="Введите ваше имя" />
      <v-button @click="enterName" title="Начать игру" class="button" />
    </div>
    <div v-else>
      <v-button
        v-if="!isPlaying"
        @click="openPopup"
        title="Как играть"
        class="button"
      />
      <v-button
        v-if="!isPlaying"
        @click="onStartGame(gameRuleWithoutBorders)"
        title="Играть без границ"
        class="button button-play"
      />
      <v-button
        v-if="!isPlaying"
        @click="onStartGame(gameRuleWithBorders)"
        title="Играть с границами"
        class="button button-play"
      />
      <v-button
        v-if="!isPlaying"
        @click="fetchRecords"
        title="Показать рекорды"
        class="button"
      />
    </div>
    <v-how-to-play-popup v-if="isShowingHowToPlayPopup" @closed="closePopup" />
    <v-playground :score="score" />
    
    <div class="footer">
      <a
        class="source-code--link"
        target="_blank"
        href="https://github.com/ekinkaradag/snake-vue3"
        >Просмотреть исходный код</a
      >
    </div>
    
    <div v-if="showRecords" class="record-list">
      <h2>Рекорды</h2>
        <table v-for="record in records" :key="record.nickname">
          <td class="playerName"> 
            {{ record.nickname }} 
          </td> 
          <td class="number"> 
            {{ record.record }} 
          </td>
        </table>
        <v-button
          @click="closeRecords"
          title="Закрыть рекорды"
          class="button closeButton"
        />
    </div>
  </div>
</template>

<script lang="ts">
import {
  computed,
  onMounted,
  onBeforeUnmount,
  type ComputedRef,
  ref,
} from "vue";
import { useStore } from "vuex";
import { areSameCoordinates, isSnake } from "@/utils/index";
import { Direction, GameRule } from "@/store/enums";
import type { ICoordinate, ISnack, ISnake } from "@/store/interfaces";

// Components
import VButton from "@/components/Button.vue";
import VHowToPlayPopup from "@/components/HowToPlayPopup.vue";
import VGrid from "@/components/Grid.vue";
import VPlayground from "@/components/Playground.vue";

const GRID_SIZE = 35;
const DIRECTION_TICKS_WITHOUT_BORDERS = {
  UP: (x: number, y: number) => ({ x, y: y <= 0 ? GRID_SIZE - 1 : y - 1 }),
  DOWN: (x: number, y: number) => ({ x, y: y >= GRID_SIZE - 1 ? 0 : y + 1 }),
  RIGHT: (x: number, y: number) => ({ x: x >= GRID_SIZE - 1 ? 0 : x + 1, y }),
  LEFT: (x: number, y: number) => ({ x: x <= 0 ? GRID_SIZE - 1 : x - 1, y }),
};
const DIRECTION_TICKS_WITH_BORDERS = {
  UP: (x: number, y: number) => ({ x, y: y - 1 }),
  DOWN: (x: number, y: number) => ({ x, y: y + 1 }),
  RIGHT: (x: number, y: number) => ({ x: x + 1, y }),
  LEFT: (x: number, y:number) => ({x: x - 1, y }),
};
const KEY_CODES_MAPPER = {
  38: Direction.UP, // ARROW_UP Key

  39: Direction.RIGHT, // ARROW_RIGHT Key

  37: Direction.LEFT, // ARROW_LEFT Key

  40: Direction.DOWN, // ARROW_DOWN Key
};

export default {
  name: "App",

  components: {
    VButton,
    VHowToPlayPopup,
    VGrid,
    VPlayground,
  },

  setup() {
    const store = useStore();
    const gameRuleWithoutBorders: ComputedRef<GameRule> = computed(
      () => GameRule.WITHOUT_BORDERS
    );
    const gameRuleWithBorders: ComputedRef<GameRule> = computed(
      () => GameRule.WITH_BORDERS
    );
    const version: ComputedRef<string> = computed(
      () => store.getters.appVersion
    );
    const isPlaying: ComputedRef<boolean> = computed(
      () => store.state.isPlaying
    );
    const currentDirection: ComputedRef<string> = computed(
      () => store.state.playground.direction
    );
    const snack: ComputedRef<ISnack> = computed(() => store.state.snack);
    const snake: ComputedRef<ISnake> = computed(() => store.state.snake);
    const snakeHead: ComputedRef<ICoordinate> = computed(
      () => store.state.snake.coordinates[0]
    );
    const snakeTail: ComputedRef<ICoordinate[]> = computed(() =>
      store.state.snake.coordinates.slice(1)
    );
    const score: ComputedRef<number> = computed(
      () => store.state.snake?.coordinates?.length - 1
    );
    const tickRate: ComputedRef<number> = computed(() => store.state.tickRate);
    const isShowingHowToPlayPopup = ref<boolean>(false);
    const hasEnteredName = ref<boolean>(false);
    const playerName = ref<string>("");
    const records = ref<Array<{nickname: string, record: number}>>([]);
    const showRecords = ref<boolean>(false);

    // Interval variable (It will only run once)
    let interval = setInterval(() => {
      clearInterval(interval);
    }, 1);

    // Generate an empty grid to be displayed at the startup
    generateGrid();

    function getRandomNumber(min: number, max: number) {
      return Math.floor(Math.random() * (max - min + 1) + min);
    }

    function getRandomCoordinate() {
      return {
        y: getRandomNumber(1, GRID_SIZE - 1),
        x: getRandomNumber(1, GRID_SIZE - 1),
      };
    }

    function getRandomSnackCoordinate() {
      let newCoordinate = getRandomCoordinate();

      if (
        snake.value.coordinates.find((snakeCellCoordinate) =>
          areSameCoordinates(snakeCellCoordinate, newCoordinate)
        )
      )
        newCoordinate = getRandomSnackCoordinate();

      return newCoordinate;
    }

    function getSnakeTail() {
      return snake.value.coordinates.slice(
        0,
        snake.value.coordinates.length - 1
      );
    }

    function generateGrid() {
      const grid: number[] = [];

      for (let i: number = 0; i < GRID_SIZE; i++) {
        grid.push(i);
      }

      store.commit("SET_GRID", grid);
    }

    function generateSnake() {
      const snake = {
        coordinates: [
          { x: Math.ceil(GRID_SIZE / 2), y: Math.ceil(GRID_SIZE / 2) },
        ],
      };

      store.commit("SET_SNAKE", snake);
    }

    function generateSnack() {
      const snack = {
        coordinate: getRandomSnackCoordinate(),
      };

      store.commit("SET_SNACK", snack);
    }

    function generateInitials() {
      resetGame();
      generateGrid();
      generateSnake();
      generateSnack();
    }

    function resetGame() {
      store.commit("RESET_GAME");
    }

    function snakeHeadTouchesTail() {
      return isSnake(snakeTail.value, snakeHead.value.x, snakeHead.value.y);
    }

    function isSnakeEating() {
      return areSameCoordinates(snakeHead.value, snack.value.coordinate);
    }

    function isSnakeOutside() {
      return (
        snakeHead.value.x >= GRID_SIZE ||
        snakeHead.value.y >= GRID_SIZE ||
        snakeHead.value.x < 0 ||
        snakeHead.value.y < 0
      );
    }

    function onChangeDirection(e: any) {
      const newDirection = KEY_CODES_MAPPER[e.keyCode];

      // Prevent scrolling if the user pushed an arrow key for navigating the snake
      if (newDirection) e.preventDefault();
      if (!newDirection || newDirection === currentDirection.value) {
        return;
      }

      store.commit("SNAKE_CHANGE_DIRECTION", newDirection);
    }

    function onTick(gameRule: GameRule) {
      if (
        snakeHeadTouchesTail() ||
        (gameRule === GameRule.WITH_BORDERS && isSnakeOutside())
      ) {
        store.commit("GAME_OVER");
        onStopGame();
        if (hasEnteredName.value) {
          sendRecord();
        }
      } else {
        store.commit("SNAKE_MOVE", {
          isSnakeEating: isSnakeEating(),
          directionTicks:
            gameRule === GameRule.WITHOUT_BORDERS
              ? DIRECTION_TICKS_WITHOUT_BORDERS
              : DIRECTION_TICKS_WITH_BORDERS,
          snakeHead: snakeHead.value,
          snakeTail: getSnakeTail(),
          snackRandomCoordinate: getRandomSnackCoordinate(),
        });
      }
    }

    function openPopup() {
      if (!isShowingHowToPlayPopup.value) isShowingHowToPlayPopup.value = true;
    }

    function closePopup() {
      if (isShowingHowToPlayPopup.value) isShowingHowToPlayPopup.value = false;
    }

    function fetchRecords() {
      // Добавить GET запрос для получения рекордов с сервера и отобразить их
      fetchRecordsFromServer();
    }

    function closeRecords() {
      showRecords.value = false;
    }

    function enterName() {
      if (playerName.value != ''){
        hasEnteredName.value = true;
      }
    }

    function sendRecord() {
      // Добавить POST запрос для отправки рекорда на сервер
      sendRecordToServer(playerName.value, score.value);
    }

    function onStartGame(gameRule: GameRule) {
      onStopGame();
      generateInitials();
      store.commit("IS_PLAYING", true);

      interval = setInterval(() => {
        onTick(gameRule);
      }, tickRate.value);
    }

    function onStopGame() {
      clearInterval(interval);
      store.commit("IS_PLAYING", false);
    }

    function sendRecordToServer(playerName, score) {
      console.log(playerName);
      const data = {
        user: playerName,
        record: score,
      };

      fetch('https://fezwer.ru/users_records', {
        method: 'POST',
        body: JSON.stringify(data),
        headers: {
          'Content-Type': 'application/json'
        }
      })
      .then(response => response.json())
      .then(data => {
        console.log(data);
      })
      .catch(error => {
        console.error('Error:', error);
      });
    }

    function fetchRecordsFromServer() {
  fetch('https://fezwer.ru/users_records', {
    method: 'GET'
  })
  .then(response => response.json())
  .then(data => {
    records.value = data;
    showRecords.value = true;
  })
  .catch(error => {
    console.error('Error:', error);
  });
}

    onMounted(() => {
      window.addEventListener("keydown", onChangeDirection);
    });

    onBeforeUnmount(() => {
      window.removeEventListener("keydown", onChangeDirection);
      clearInterval(interval);
    });

    return {
      version,
      gameRuleWithoutBorders,
      gameRuleWithBorders,
      isPlaying,
      score,
      isShowingHowToPlayPopup,
      hasEnteredName,
      playerName,
      records,
      showRecords,
      openPopup,
      closePopup,
      onStartGame,
      onStopGame,
      fetchRecords,
      enterName,
      closeRecords,
    };
  },
};
</script>
<style lang="postcss" scoped>
.page {
  width: 100%;
  text-align: center;
}

.button {
  margin: 0 10px;
  margin-bottom: 20px;
}

.button-play {
  width: 190px;
}

.title {
  color: rgb(0, 199, 0);
  margin-left: 30px;
  letter-spacing: 30px;
  text-shadow: 1px 1px 1px darkgreen, -1px 1px 1px darkgreen,
    1px -1px 1px darkgreen, -1px -1px 1px darkgreen, 0 0 64px lightgreen,
    0 0 64px lightgreen;
  font-size: 64px;
  font-family: "Courier", monospace;
  font-weight: bold;
}

.header {
  width: 100%;
  display: flex;
  flex-direction: row;
  color: gray;
  font-family: sans-serif;
}

.disclaimer {
  flex: 1;
  text-align: end;
}

.social-links {
  margin-bottom: 10px;
}

.source-code--link {
  background-color: gray;
  color: black !important;
  font-size: 14px;
  font-weight: 800;
  border: solid gray;
  border-top-width: 2.6px;
  border-bottom-width: 2.6px;
  border-left-width: 16.6px;
  border-right-width: 16.6px;
  border-radius: 5px;
  text-decoration: none;
  font-family: Inter, -apple-system, system-ui, "Segoe UI", Helvetica, Arial,
    sans-serif;
}

.footer {
  margin-top: 20px;
}
</style>
 
