<template>
  <h1>Wordle Clone</h1>

  <div class="toolbar">
    <button class="button-normal" @click="gameOver" v-show="rowCurrent > 0">Me rindo</button>
  </div>

  <!-- cuadricula -->
  <div class="box-content df-c">
    <div class="box-row" v-for="row in boxSetting.row" :key="row">
      <div v-for="column in boxSetting.column" :key="row + column" class="box df-c" :class="{
  correct: isCorrect(row, column),
  incorrect: isFail(row, column),
  hot: isHot(row, column)
}" :id="getIndexBox(row, column)">
        {{ letterBox[getIndexBox(row, column)]?.key }}</div>
    </div>
    <!-- Ventana de mensajes -->
    <div class="window-message window-center df-c" v-show="bShowMessage">{{ messageText }}</div>
    <!-- Ventana de terminar el juego -->
    <div class="ligh-box window-center" v-show="windowEndgame.show">
      <div class="window-endgame window-center">
        <div class="window-endgame-title">{{ windowEndgame.title }}</div>
        <div class="window-endgame-subtitle">La respuesta fue:</div>
        <div class="window-endgame-resp">{{ windowEndgame.resp }}</div>
        <button class="button-secondary" @click="OnResetGame">REINICIAR</button>
      </div>
    </div>
  </div>

  <div class="info-game">
    <h3>Juego Wordle Clone: Adivina la Palabra Española Oculta</h3>
    <p>
      Las reglas son muy simples: debes adivinar la palabra oculta en español en 6 intentos. Para comenzar, simplemente
      escriba cualquier palabra en la primera línea. Si la letra se adivina correctamente y está en el lugar correcto,
      se
      marcará en verde, si la letra está en la palabra, pero en el lugar equivocado, en amarillo, y si la letra no está
      en
      la palabra, permanecerá gris. ¿Puedes adivinar la palabra española oculta en 6 intentos?
    </p>
  </div>
</template>

<script>
import { reactive } from 'vue'
// Ultradiccionario 
import words from 'an-array-of-spanish-words'
let words_infinito = words.filter(d => d.length == 5)
let words_nomames = words_infinito.filter(d => d.includes('ñ') == false).sort((a, b) => Math.random() - 0.4)

const bDebugLog = false;

// Usamos la lista de palabras para verificar la del usuario, esto ayuda a no usar letras unicas para descubrir las ganadoras o palabras inventadas.
// ** Trata de tener muchas palabras para utilizar esto. **
const wordUserWriteCheckListExist = true;

const boxState = { normal: 0, correct: 1, fail: 2, hot: 3 };
const boxSetting = { column: 5, row: 6 };

const wordsList = words_nomames;
const message1 = "Palabra no encontrada";
const message2 = "Una palabra demasiado corta";
const message3 = "¡Adivina la primera palabra!";
const message4 = "¡Ganaste!";
const message5 = "¡Perdiste!";

const messageTimeShowDefault = 1000;

export default {
  name: 'App',
  data() {
    return {
      windowEndgame: { show: false, title: '', resp: '' },
      // other
      boxSetting: boxSetting,
      messageText: "",
      keyDownEventLock: false,
      bShowMessage: false,
      rowCurrent: 0,
      // word
      wordIndexList: 0,
      // letter
      letterBox: [],
      letterIndex: 0,
      letterJumpRow: 0,
    }
  },

  methods: {
    OnResetGame() {
      this.hideDialogEndGame();

      this.letterBox = [];
      this.letterIndex = 0;
      this.letterJumpRow = 0;
      this.rowCurrent = 0;
      this.wordIndexList = Math.floor(Math.random() * wordsList.length);

      this.printLog(`IdWord: ${this.wordIndexList} Word: ${wordsList[this.wordIndexList]}`);
    },
    OnKeyDown(event) {
      const key = event.key
      if (this.keyDownEventLock) return false;
      if (key == 'Enter') {
        this.checkRowWord();
        return false;
      } else if (key == 'Backspace') {
        this.backspaceAndLetterClear();
        return false;
      }
      this.writeInBoxIndex(key);
    },
    printLog(text) {
      if (bDebugLog) {
        console.log(text);
      }
    },
    gameOver() {
      this.showDialogEndGame(message5);
    },
    checkRowWord() {
      let lFoundletter = false;
      let word_user_flat = '';

      const words = wordsList[this.wordIndexList].split('');
      const word_user = this.letterBox.slice(this.letterJumpRow, this.getLetterIndex());

      // Verificamos que tenemos la cantidad correcta de letras
      if (word_user.length != boxSetting.column) {
        this.showMessage(message2);
        return false;
      }

      // verificamos si tenemos todas las letras
      for (const words_index in words) {
        let index = parseInt(words_index);
        let iStatus = boxState.fail;
        word_user_flat += word_user[index].key;
        if (words[index] === word_user[index].key) {
          lFoundletter = true;
          iStatus = boxState.correct;
          words[index] = '*'
        }
        this.SetLetterStatusActiveRow(index, iStatus);
      }

      //
      // Está en la palabra pero en el lugar equivocado.
      // M I S I L
      // R A D I O
      // prueba 8 - :) !!!

      // verificamos las letras boxState.fail, para
      // ver si una de ellas se encuentra en otra posicion.
      for (const wordusr_index in word_user) {
        let index = parseInt(wordusr_index);
        if (word_user[wordusr_index].state == boxState.fail) {
          let iStatus = boxState.fail;
          const r = words.findIndex(w => w == word_user[wordusr_index].key);
          if (r != -1)
            this.SetLetterStatusActiveRow(index, boxState.hot);
        }
      }

      // Si utilizamos la lista de palabras para comprobar
      // si existe alguna igual a la que usuario escribio.
      if (wordUserWriteCheckListExist) {
        const id = wordsList.findIndex(w => w == word_user_flat);
        lFoundletter = id != -1;
      }
      if (lFoundletter == false) {
        // reiniciamos los estados cuando no encontramos coincidencias
        // Y mostramos un mensaje que se quita segun messageTimeShowDefault.
        word_user.forEach(wu => wu.state = 0);
        this.showMessage(message1);
      } else {

        // comprobamos si todos los estados de word_user son correct.
        const isCorrectAll = word_user.filter(wu => wu.state == boxState.correct)
        if (isCorrectAll.length == word_user.length) {
          this.showDialogEndGame(message4);
          return;
        }

        // comprobamos si estamos en la ultima fila
        if (this.rowCurrent >= (boxSetting.row - 1)) {
          this.showDialogEndGame(message5);
          return;
        }

        // movemos el cursor al inicio
        this.letterStart();

        // nos movemos hacia la siguiente fila
        this.letterNextRow();
      }
    },
    SetLetterStatusActiveRow(i, state) {
      this.letterBox[(this.letterJumpRow + i)].state = state
    },
    GetLetterStatusActiveRow(i) {
      const state = this.letterBox[(this.letterJumpRow + i)]?.state;
      return state;
    },
    writeInBoxIndex(key) {
      const new_key = key.toLocaleLowerCase().replace(/[^a-z]/g, '')
      if (new_key.length == 1 && key != ' ' && this.letterIndex < boxSetting.column) {
        this.letterSetValue(new_key);
        this.letterMoveRight();
      }
    },
    backspaceAndLetterClear() {
      this.letterMoveLeft();
      this.letterSetValue();
    },
    letterSetValue(key = '', state = boxState.normal) {
      this.letterBox[this.getLetterIndex()] = reactive({ "key": key, "state": state });
    },
    letterMoveLeft() {
      this.letterIndex--;
      this.letterIndex = Math.max(this.letterIndex, 0)
    },
    letterMoveRight() {
      this.letterIndex++;
      //this.letterIndex = Math.min(this.letterIndex, boxSetting.column)
    },
    letterStart() {
      this.letterIndex = 0;
    },
    letterNextRow() {
      this.letterJumpRow += 5;
      // tenemos un row current realista, para saber en que
      // file esta el jugador escribiendo.
      this.rowCurrent += 1;
    },
    getLetterIndex() {
      return this.letterIndex + this.letterJumpRow;
    },
    getIndexBox(row, column) {
      const id = (row - 1) * boxSetting.column + column;
      return id - 1;
    },
    isCorrect(row, column) {
      const id = this.getIndexBox(row, column);
      return this.letterBox[(id)]?.state == boxState.correct
    },
    isFail(row, column) {
      const id = this.getIndexBox(row, column);
      return this.letterBox[(id)]?.state == boxState.fail
    },
    isHot(row, column) {
      const id = this.getIndexBox(row, column);
      return this.letterBox[(id)]?.state == boxState.hot
    },
    showMessage(msg, time = messageTimeShowDefault) {
      this.messageText = msg;
      this.keyDownEventLock = this.bShowMessage = true;
      setTimeout(() => { this.keyDownEventLock = this.bShowMessage = false; }, time);
    },
    showDialogEndGame(title = message4) {
      this.keyDownEventLock = true;
      this.windowEndgame.show = true;
      this.windowEndgame.title = title;
      this.windowEndgame.resp = wordsList[this.wordIndexList];
    },
    hideDialogEndGame() {
      this.windowEndgame.show = false;
      this.keyDownEventLock = false;
    }
  },

  created() {
    this.boxCount = 0;
    window.addEventListener('keydown', this.OnKeyDown);
    this.OnResetGame();
    this.showMessage(message3, 2000);
  }
}
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Montserrat:wght@500;700&family=Roboto:ital,wght@0,100;0,300;0,400;0,700;0,900;1,100;1,300;1,400;1,500;1,700;1,900&display=swap');

body {
  margin: 0;
  padding: 0;
}

#app {
  font-family: "Roboto", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}

h1 {
  font-family: 'Roboto', sans-serif;
  user-select: none;
}

.df-c {
  display: flex;
  justify-content: center;
  align-items: center;
}

.wrap {
  flex-wrap: wrap;
}

.box-content {
  width: 500px;
  margin: auto;
  flex-wrap: wrap;
  position: relative;
}

.box-row {
  display: flex;
}

.box {
  width: 50px;
  height: 50px;

  background-color: #fbfcff;

  border: 2px solid #dee1e9;
  border-radius: 5px;

  font-size: 28px;

  margin: 3px;

  font-family: 'Montserrat', sans-serif;
  font-weight: bold;
  text-transform: uppercase;

  user-select: none;
}

.box.correct {
  background-color: #79b851;
}

.box.incorrect {
  background-color: #a4aec4;
}

.box.hot {
  background-color: #f3c237;
}

.box:is(.correct, .incorrect, .hot) {
  border-color: transparent;
  color: white;
  box-shadow: inset 0px 0px 3px 0px rgba(0, 0, 0, 0.067);
}

.window-center {
  position: absolute;
  top: 0;
  bottom: 0;
  left: 0;
  right: 0;
  margin: auto;
  z-index: 9;
}

.window-message {
  margin: auto;
  width: 300px;
  height: 100px;
  background-color: white;
  border-radius: 10px;
  box-shadow: 0px 0px 20px 0px rgb(0 0 0 / 34%);
  font-size: 20px;
  font-weight: 700;
  font-family: 'Montserrat', sans-serif;
  padding: 10px;
}

.button-secondary {
  background-color: #57ac57;
  color: white;
  font-weight: bold;
  font-family: 'Montserrat';
  padding: 15px;
  border-radius: 5px;
  border: 0px;
}

.button-secondary:hover {
  cursor: pointer;
  background-color: #478c47;
}

.button-normal {
  background-color: #ebedf3;
  color: #6a6a6a;
  font-weight: bold;
  font-family: 'Montserrat';
  padding: 9px;
  border-radius: 5px;
  border: 0px;
}

.button-normal:hover {
  cursor: pointer;
  background-color: #c9cad0;
}

.toolbar {
  height: 50px;
  width: 295px;
  margin: auto;
  text-align: left;
}

.ligh-box {
  width: 100%;
  height: 100%;
  background-color: rgba(249, 249, 249, 0.218);
  position: fixed;
}

.window-endgame {
  width: 250px;
  height: 200px;
  border-radius: 9px;
  overflow: hidden;
  box-shadow: 0px 0px 50px 0px rgb(0 0 0 / 15%);
  font-family: 'Montserrat';
  background-color: #fff;
}

.window-endgame-title {
  background-color: #e5ecff;
  padding: 10px;
  font-size: 20px;
  margin-bottom: 5px;
}

.window-endgame-subtitle {
  font-size: 14px;
}

.window-endgame-resp {
  padding: 10px;
  border: 1px dashed #75819e;
  width: 80px;
  background-color: #f1f3f9;
  border-radius: 5px;
  margin: auto;
  margin-top: 10px;
  margin-bottom: 10px;
  font-weight: bold;
  font-size: 22px;
  text-transform: uppercase;
}

.info-game {
  background: #ecf0e2;
  padding: 20px;
  margin-top: 30px;
}
.info-game p {
  font-size: 15px;
}
</style>
