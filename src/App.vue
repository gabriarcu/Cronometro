<template>
  <div id="app">
    <img class="img" src="./assets/cronometro.png" />
    <!-- inserire visualizzazione per il timer -->
    <span class="timer"
      >{{ zfill(this.hour) }}:{{ zfill(this.min) }}:{{ zfill(this.sec) }}</span
    >
    <div class="btns">
      <!-- aggiungere event listener al click al bottone, il click deve azionare il metodo play -->
      <button v-if="this.timer === null" class="btn btn-margin" @click="play">
        <!-- Inserire testo per il bottone che cambi dinamicamente da VIA a PAUSA quando è in corso un timer -->
        <!-- {{this.timer===null ? "VIA": "PAUSA"}} -->
        VIA
      </button>
      <button v-if="this.timer != null" class="btn btn-margin" @click="stop">
        <!-- Inserire testo per il bottone che cambi dinamicamente da VIA a PAUSA quando è in corso un timer -->
        <!-- {{this.timer===null ? "VIA": "PAUSA"}} -->
        PAUSA
      </button>
      <!-- aggiungere event listener al click al bottone, il click deve azionare il metodo clear -->
      <button class="btn btn-margin" @click="clear">RESET</button>
    </div>
    <!-- aggiungere visualizzazione condizionale di questa sezione -->
    <div class="interval">
      <ul>
        <!-- bisogna creare un li per ogni interval in intervalList -->
        <li v-for="timer in intervalList" :key="timer">
          <!-- visualizzare tempo trascorso -->
          TEMPO: {{ timer }}
        </li>
      </ul>
      <!-- aggiungere event listener al click al bottone, il click deve azionare il metodo clearIntervalList -->
      <button class="btn" @click="clearIntervalList">SVUOTA STORICO</button>
    </div>
  </div>
</template>

<script>
export default {
  name: "App",
  data() {
    return {
      sec: 0,
      min: 0,
      hour: 0,
      timer: null,
      intervalList: [],
    };
  },
  methods: {
    zfill(number) {
      // metodo che aggiunge un padding al numero e lo restituisce es: 1 --> 01
      return number.toString().padStart(2, 0);
    },
    play() {
      // aggiungere metodo per fare partire un intervallo con delay di 1 secondo e aggiungerlo alla variabile timer se il timer è uguale a null,
      // altrimenti rimuovere l'interval con clearInterval(this.timer) e resettare il timer a null
      // nel caso in cui il timer sia uguale a null bisogna chiamare il metodo this.playing() all'inizio sia dentro all'intervallo
      // nel caso in cui il timer sia null allora bisogna chiamare il metodo this.pause()

      this.playing();
      this.timer = setInterval(this.playing, 1000);
    },
    stop() {
      clearInterval(this.timer);
      this.timer = null;
      this.pause();
    },

    playing() {
      // devo aumentare i secondi in modo che 60 secondi mi aumentino i minuti di 1 e resettino i secondi
      // e che 60 minuti mi aumentino le ore di 1 resettando poi i minuti
      this.sec++;
      if (this.sec >= 60) {
        this.sec = 0;
        this.min++;
      }
      if (this.min >= 60) {
        this.min = 0;
        this.hour++;
      }
    },
    pause() {
      // devo aggiungere l'orario formattato con `${this.zfill(this.hour)}:${this.zfill(this.min)}:${this.zfill(this.sec)}` a intervalList
      const formattedTimer = `${this.zfill(this.hour)}:${this.zfill(
        this.min
      )}:${this.zfill(this.sec)}`;
      this.intervalList.push(formattedTimer);
    },
    clear() {
      // se il timer è diverso da null allora devo chiamare il metodo clearInterval(this.timer) e resettare il timer a null
      // devo poi settare i secondi, minuti, ore a zero e chiamare il metodo this.clearIntervalList()
      if (this.timer != null) {
        clearInterval(this.timer);
        this.timer = null;
      }
      this.sec = 0;
      this.min = 0;
      this.hour = 0;
    },
    clearIntervalList() {
      // bisogna svuotare la intervalList
      this.intervalList = [];
    },
  },
};
</script>

<style>
#app {
  display: flex;
  width: 100%;
  align-items: center;
  justify-content: center;
  flex-direction: column;
}
.img {
  width: 420px;
  height: 420px;
  padding-top: 100px;
}
.timer {
  color: #fff;
  font-size: 70px;
  margin-top: -210px;
}
.btns {
  margin-top: 155px;
  display: flex;
}
.btn {
  -webkit-user-select: none;
  -moz-user-select: none;
  width: 150px;
  background-color: #fff;
  font-size: 20px;
  border: none;
  border-radius: 5px;
  text-align: center;
  padding: 6px;
  cursor: pointer;
  transition: all ease-in-out 0.3s;
}
.btn-margin {
  margin: 0px 7px;
}
.btn:hover {
  opacity: 0.8;
}
.interval {
  color: #fff;
  flex: 1;
  width: 420px;
  margin-top: 15px;
}
.interval ul {
  text-align: center;
}
.interval ul li {
  list-style: none;
  background-color: rgb(70, 70, 70);
  padding: 15px;
  margin-bottom: 10px;
}
</style>
