<template>
  <div id="app" class="container">
    <div v-if="!inBattle && !selectingPokemons && !battleStarted" class="selection battle-background">
      <div>
        <img class="imagentitulo" src="https://upload.wikimedia.org/wikipedia/commons/thumb/9/98/International_Pok%C3%A9mon_logo.svg/640px-International_Pok%C3%A9mon_logo.svg.png" alt="">
      </div>
      <div class="rounds">
        <button @click="selectRounds(3)">3 Rondas</button>
        <button @click="selectRounds(6)">6 Rondas</button>
        <button @click="selectRounds(8)">8 Rondas</button>
      </div>
    </div>

    <div v-if="selectingPokemons && !battleStarted" class="pokemon-selection battle-background">
      <h1>Jugador {{ currentPlayer }}: Selecciona tus Pokémon</h1>
      <div class="pokemon-container">
        <div class="pokemon-card"
             v-for="pokemon in allPokemons"
             :key="pokemon.id"
             @click="selectPokemon(pokemon)"
             :style="{ backgroundColor: getPokemonTypeColor(pokemon) }">
          <p class="pokemon-name">{{ pokemon.name }}</p>
          <img :src="pokemon.sprites?.front_default" alt="Pokémon" class="pokemon-image animated" />
        </div>
      </div>
      <div>
        <p>Pokémon seleccionados:</p>
        <ul>
          <li v-for="pokemon in selectedPokemonsPlayer1" :key="pokemon.id">
            <span :style="{ color: getPokemonTypeColor(pokemon) }">{{ pokemon.name }}</span>
            <select v-model="selectedStatPlayer1[pokemon.id]" @change="setFightingPokemon(pokemon)">
              <option disabled value="">Selecciona Estadística</option>
              <option v-for="stat in pokemon.stats" :key="stat.stat.name" :value="stat.stat.name">{{ stat.stat.name }}</option>
            </select>
          </li>
        </ul>
      </div>
      <button @click="nextPlayer" class="next-button" v-if="currentPlayer === 1 && selectedPokemonsPlayer1.length === rounds">
        Siguiente Jugador
      </button>
      <button @click="nextPlayer" class="next-button" v-if="currentPlayer === 2 && selectedPokemonsPlayer2.length === rounds">
        Siguiente Jugador
      </button>
      <button @click="startBattle" class="start-button" v-if="currentPlayer === 2 && selectedPokemonsPlayer2.length === rounds">
        Iniciar Batalla
      </button>
      <button @click="reset" class="reset-button">Volver a seleccionar rondas</button>
    </div>

    <div v-if="battleStarted" class="battles battle-background">
      <h1>Batalla Pokémon - Ronda {{ currentRound + 1 }} de {{ rounds }}</h1>
      <div v-if="loading" class="loading">Cargando...</div>
      <div v-else class="battle-arena">
        <!-- Pokémon de Jugador 1 -->
        <div class="player-pokemon">
          <h2>Jugador 1</h2>
          <div :style="{ backgroundColor: getPokemonTypeColor(currentPokemons[0]) }" class="pokemon-card battle-pokemon">
            <p class="pokemon-name">{{ currentPokemons[0].name }}</p>
            <img :src="currentPokemons[0].sprites?.front_default" alt="Pokémon" class="pokemon-image animated" />
            <div class="stats">
              <p>HP: {{ currentPokemons[0].stats[0].base_stat }}</p>
              <p>Ataque: {{ currentPokemons[0].stats[1].base_stat }}</p>
              <p>Defensa: {{ currentPokemons[0].stats[2].base_stat }}</p>
            </div>
          </div>
        </div>

        <div class="vs">VS</div>

        <div class="player-pokemon">
          <h2>Jugador 2</h2>
          <div :style="{ backgroundColor: getPokemonTypeColor(currentPokemons[1]) }" class="pokemon-card battle-pokemon">
            <p class="pokemon-name">{{ currentPokemons[1].name }}</p>
            <img :src="currentPokemons[1].sprites?.front_default" alt="Pokémon" class="pokemon-image animated" />
            <div class="stats">
              <p>HP: {{ currentPokemons[1].stats[0].base_stat }}</p>
              <p>Ataque: {{ currentPokemons[1].stats[1].base_stat }}</p>
              <p>Defensa: {{ currentPokemons[1].stats[2].base_stat }}</p>
            </div>
          </div>
        </div>
      </div>

      <div class="buttons">
        <button @click="determineWinner" class="next-button">Ver resultado de la ronda</button>
        <button @click="nextRound" class="next-button" v-if="winner">Siguiente Ronda</button>
        <button @click="reset" class="reset-button">Volver a seleccionar rondas</button>
      </div>

      <div v-if="winner" class="winner">¡Ganador de la ronda: {{ winner.name }}!</div>
    </div>

    <div v-if="battleEnded" class="final-winner">
      <h1>¡Ganador de la batalla: Jugador {{ finalWinner }}!</h1>
      <button @click="reset" class="reset-button">Volver a iniciar</button>
    </div>
  </div>
</template>

<script setup>

import axios from 'axios';
import { ref, reactive, onMounted } from 'vue'; 

export default {
  setup() {
    const rounds = ref(null);
    const allPokemons = ref([]);
    const selectedPokemonsPlayer1 = ref([]);
    const selectedPokemonsPlayer2 = ref([]);
    const loading = ref(false);
    const selectingPokemons = ref(false);
    const currentPlayer = ref(1);
    const currentRound = ref(0);
    const winner = ref(null);
    const currentPokemons = ref([]);
    const battleStarted = ref(false);
    const battleEnded = ref(false);
    const finalWinner = ref(null);
    const selectedStatPlayer1 = ref({});
    const selectedStatPlayer2 = ref({});

    const fetchAllPokemons = async () => {
      loading.value = true;
      const response = await axios.get('https://pokeapi.co/api/v2/pokemon?limit=150');
      const pokemons = response.data.results;
      allPokemons.value = await Promise.all(pokemons.map((pokemon, index) => getPokemonDetails(pokemon.url, index)));
      loading.value = false;
    };

    const getPokemonDetails = async (url, index) => {
      const response = await axios.get(url);
      return response.data;
    };

    const selectRounds = (numRounds) => {
      rounds.value = numRounds;
      selectingPokemons.value = true;
      fetchAllPokemons();
    };

    const selectPokemon = (pokemon) => {
      if (currentPlayer.value === 1 && selectedPokemonsPlayer1.value.length < rounds.value) {
        selectedPokemonsPlayer1.value.push(pokemon);
      } else if (currentPlayer.value === 2 && selectedPokemonsPlayer2.value.length < rounds.value) {
        selectedPokemonsPlayer2.value.push(pokemon);
      }
    };

    const setFightingPokemon = (pokemon) => {

      console.log(`Pokemon seleccionado para pelear: ${pokemon.name} con la estadística: ${selectedStatPlayer1[pokemon.id]}`);
    };

    const nextPlayer = () => {
      if (currentPlayer.value === 1) {
        currentPlayer.value = 2;
      } else {
        currentPlayer.value = 1;
        selectingPokemons.value = false;
        startBattle();
      }
    };

    const startBattle = () => {
      battleStarted.value = true;
      currentRound.value = 0;
      currentPokemons.value = [
        selectedPokemonsPlayer1.value[currentRound.value],
        selectedPokemonsPlayer2.value[currentRound.value],
      ];
      determineWinner();
    };

    const determineWinner = () => {
      const pokemon1 = currentPokemons.value[0];
      const pokemon2 = currentPokemons.value[1];

      const totalStats1 = pokemon1.stats.reduce((sum, stat) => sum + stat.base_stat, 0);
      const totalStats2 = pokemon2.stats.reduce((sum, stat) => sum + stat.base_stat, 0);

      if (totalStats1 > totalStats2) {
        winner.value = pokemon1;
      } else if (totalStats2 > totalStats1) {
        winner.value = pokemon2;
      } else {
        winner.value = { name: "Empate" };
      }
    };

    const nextRound = () => {
      currentRound.value++;
      if (currentRound.value < rounds.value) {
        currentPokemons.value = [
          selectedPokemonsPlayer1.value[currentRound.value],
          selectedPokemonsPlayer2.value[currentRound.value],
        ];
        winner.value = null; 
        determineWinner();
      } else {
        endBattle();
      }
    };

    const endBattle = () => {
      battleEnded.value = true;
      finalWinner.value = winner.value.name; 
    };

    const reset = () => {
      rounds.value = null;
      selectedPokemonsPlayer1.value = [];
      selectedPokemonsPlayer2.value = [];
      loading.value = false;
      selectingPokemons.value = false;
      currentPlayer.value = 1;
      currentRound.value = 0;
      winner.value = null;
      currentPokemons.value = [];
      battleStarted.value = false;
      battleEnded.value = false;
      finalWinner.value = null;
    };

    const getPokemonTypeColor = (pokemon) => {
      const types = pokemon.types.map(type => type.type.name);

      const colors = {
        fire: '#F08030',
        water: '#6890F0',
        grass: '#78C850',
        electric: '#F8D030',
 
      };
      return colors[types[0]] || '#FFF'; 
    };

    onMounted(fetchAllPokemons);

    return {
      rounds,
      allPokemons,
      selectedPokemonsPlayer1,
      selectedPokemonsPlayer2,
      loading,
      selectingPokemons,
      currentPlayer,
      currentRound,
      winner,
      currentPokemons,
      battleStarted,
      battleEnded,
      finalWinner,
      selectedStatPlayer1,
      selectedStatPlayer2,
      selectRounds,
      selectPokemon,
      nextPlayer,
      startBattle,
      determineWinner,
      nextRound,
      endBattle,
      reset,
      getPokemonTypeColor,
    };
  }
};
</script>

<style>

.selection, .pokemon-selection, .battles, .final-winner {
  backdrop-filter: none; 
  padding: 20px;
  margin-bottom: 20px;
  border: 2px solid white;
}


#app {
  text-align: center;
  background-image: url('https://i.pinimg.com/originals/bd/b4/0a/bdb40a570ce75676fd8b69f4f6d52851.gif'); 
  background-position: center;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 20px;
  color: rgba(0, 0, 0, 0.752);
  background-repeat: no-repeat;
  width: 100%;
  background-size: cover;
}
.rounds button, .next-button, .start-button, .reset-button {
  background: rgb(197,26,255);
  background: radial-gradient(circle, rgba(197,26,255,1) 0%, rgba(190,0,0,0.8757878151260504) 100%);
  color: rgb(0, 0, 0); 
  border: none;
  border-radius: 5px;
  padding: 30px 60px;
  font-size: 24px; 
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  margin-left: 30px;
}

.rounds button:hover, .next-button:hover, .start-button:hover, .reset-button:hover {
  background-color: rgba(0, 123, 255, 1); 
}



.pokemon-container {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-around;
  margin-top: 20px;
}

.pokemon-card {
  margin: 30px;
  padding: 30px;
  border-radius: 10px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  cursor: pointer;
  width: 200px;
}


.battles, .final-winner {
  background-image: url('https://img.freepik.com/vector-gratis/fondo-efecto-zoom-degradado_23-2149737565.jpg');
  background-size: cover;
  background-position: center;
  padding: 20px;
}

.battle-background {
  background-color:transparent;
  backdrop-filter: blur(10px);
  border-radius: 10px;
  padding: 80px;
}

.battle-arena {
  display: flex;
  justify-content: center;
  align-items: flex-start;
  margin-top: 20px;
}

.vs {
  font-size: 36px;
  font-weight: bold;
  margin: 0 40px;
}

.player-pokemon {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.stats {
  background-color: #a09b9b80;
  padding: 40px;
  border-radius: 8px;
  margin-top: 10px;
  font-size: 20px;
}

.next-button,
.start-button,
.reset-button {
  margin: 10px;
  padding: 15px 30px;
  font-size: 20px;
  background-color: #007bff;
  color: rgb(0, 0, 0);
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.reset-button:hover,
.next-button:hover,
.start-button:hover {
  background-color: #0056b3;
}

.loading {
  font-size: 24px;
  color: #555;
}

.winner {
  font-size: 24px;
  font-weight: bold;
  color: green;
}

.final-winner {
  font-size: 28px;
  font-weight: bold;
  color: #ff4500;
  margin-top: 20px;
}

.buttons {
  margin-top: 20px;
  display: flex;
  justify-content: center;
  gap: 20px;
}

.battle-pokemon {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.animated {
  animation: pulse 1.5s infinite;
}

@keyframes pulse {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.05);
  }
  100% {
    transform: scale(1);
  }
}

.pokemon-name {
  font-size: 18px;
  margin-bottom: 5px;
}

.pokemon-image {
  width: 100px;
  height: 100px;
  border-radius: 50%;
  animation: pulse 1.5s infinite;
}

h1, h2, .pokemon-name, .vs, .winner, .final-winner {
  font-family: 'Press Start 2P', cursive;
  font-size: 20px; 
  color:  #5f0e5f;
}


h1 {
  font-size: 60px; 
  text-transform: uppercase;
  letter-spacing: 2px;
}

h2 {
  font-size: 50px;
  letter-spacing: 1px;
}


.pokemon-name, .vs {
  font-size: 22px;
  text-transform: uppercase;
  letter-spacing: 1px;
  margin-top: 4%;
}

.winner, .final-winner {
  font-size: 30px;
  text-transform: uppercase;
  color: #00ff2a; 
}

#bgVideo {
  object-fit: cover;
  z-index: -1;
  }
.imageninicio{
  width: 60%;
  padding: 2%;
  border-radius: 10%;
}
.imagentitulo {
  background-color: transparent; 
  display: inline-block;
  margin: 0;
  padding: 0; 
  margin: 10%;
}

</style>
