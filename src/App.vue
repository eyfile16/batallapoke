<template>
  <div id="app" class="container">
    <!-- Selección de Rondas -->
    <div v-if="!selectingPokemons && !battleStarted && !battleEnded" class="selection battle-background">
      <div>
        <img class="imagentitulo" src="https://upload.wikimedia.org/wikipedia/commons/thumb/9/98/International_Pok%C3%A9mon_logo.svg/640px-International_Pok%C3%A9mon_logo.svg.png" alt="Pokémon Logo">
      </div>
      <div class="rounds">
        <button @click="selectRounds(3)">3 Rondas</button>
        <button @click="selectRounds(6)">6 Rondas</button>
        <button @click="selectRounds(8)">8 Rondas</button>
      </div>
    </div>


    <div v-if="selectingPokemons && !battleStarted && !battleEnded" class="pokemon-selection battle-background">
      <h1>Jugador {{ currentPlayer }}: Selecciona tus Pokémon</h1>
      <div class="selection-container">

        <div class="selected-pokemons">
          <h2>Jugador 1</h2>
          <ul>
            <li v-for="pokemon in selectedPokemonsPlayer1" :key="pokemon.id">
              <span :style="{ color: getPokemonTypeColor(pokemon) }">{{ capitalize(pokemon.name) }}</span>
              <span v-if="selectedStatPlayer1[pokemon.id]"> - {{ capitalize(selectedStatPlayer1[pokemon.id]) }}</span>
            </li>
          </ul>
        </div>


        <div class="pokemon-container">
          <div class="pokemon-card"
               v-for="pokemon in allPokemons"
               :key="pokemon.id"
               @click="selectPokemon(pokemon)"
               :style="{ backgroundColor: getPokemonTypeColor(pokemon) }"
               :class="{ selected: isPokemonSelected(pokemon) }"> 
            <p class="pokemon-name">{{ capitalize(pokemon.name) }}</p>
            <img :src="pokemon.sprites.front_default" alt="Pokémon" class="pokemon-image animated" />


            <div v-if="currentPlayer === 1 && isPokemonSelected(pokemon)">
              <label>Selecciona Estadística:</label>
              <select
                v-model="selectedStatPlayer1[pokemon.id]"
                @change="handleStatChange(pokemon.id, 1)"
              >
                <option disabled value="">Selecciona Estadística</option>
                <option v-for="stat in pokemon.stats" :key="stat.name" :value="stat.name">{{ capitalize(stat.name) }}</option>
              </select>
            </div>

            <!-- Mostrar estadística asignada para Jugador 2 -->
            <div v-if="currentPlayer === 2 && isPokemonSelected(pokemon)">
              <label>Estadística Asignada:</label>
              <span>{{ capitalize(selectedStatPlayer2[pokemon.id]) }}</span>
            </div>
          </div>
        </div>


        <div class="selected-pokemons">
          <h2>Jugador 2</h2>
          <ul>
            <li v-for="pokemon in selectedPokemonsPlayer2" :key="pokemon.id">
              <span :style="{ color: getPokemonTypeColor(pokemon) }">{{ capitalize(pokemon.name) }}</span>
              <span v-if="selectedStatPlayer2[pokemon.id]"> - {{ capitalize(selectedStatPlayer2[pokemon.id]) }}</span>
            </li>
          </ul>
        </div>
      </div>


      <div class="buttons">
        <button
          @click="nextPlayer"
          class="next-button"
          v-if="currentPlayer === 1 && selectedPokemonsPlayer1.length === rounds && areAllStatsSelected(selectedStatPlayer1)"
        >
          Siguiente Jugador
        </button>
        <button
          @click="startBattle"
          class="start-button"
          v-if="currentPlayer === 2 && selectedPokemonsPlayer2.length === rounds && areAllStatsSelected(selectedStatPlayer2)"
        >
          Iniciar Batalla
        </button>
        <button @click="reset" class="reset-button">Volver a seleccionar rondas</button>
      </div>
    </div>


    <div v-if="battleStarted" class="battles battle-background">
      <h1>Batalla Pokémon - Ronda {{ currentRound + 1 }} de {{ rounds }}</h1>
      <div v-if="loading" class="loading">Cargando...</div>
      <div v-else class="battle-arena">
        <div class="player-pokemon">
          <h2>Jugador 1</h2>
          <div :style="{ backgroundColor: getPokemonTypeColor(currentPokemons[0]) }" class="pokemon-card battle-pokemon">
            <p class="pokemon-name">{{ capitalize(currentPokemons[0].name) }}</p>
            <img :src="currentPokemons[0].sprites.front_default" alt="Pokémon" class="pokemon-image animated" />
            <div class="stats">
              <p v-if="selectedStatPlayer1[currentPokemons[0].id]">
                {{ capitalize(selectedStatPlayer1[currentPokemons[0].id]) }}: {{ getStatValue(currentPokemons[0], selectedStatPlayer1[currentPokemons[0].id]) }}
              </p>
            </div>
          </div>
        </div>

        <div class="vs">VS</div>

        <div class="player-pokemon">
          <h2>Jugador 2</h2>
          <div :style="{ backgroundColor: getPokemonTypeColor(currentPokemons[1]) }" class="pokemon-card battle-pokemon">
            <p class="pokemon-name">{{ capitalize(currentPokemons[1].name) }}</p>
            <img :src="currentPokemons[1].sprites.front_default" alt="Pokémon" class="pokemon-image animated" />
            <div class="stats">
              <p v-if="selectedStatPlayer2[currentPokemons[1].id]">
                {{ capitalize(selectedStatPlayer2[currentPokemons[1].id]) }}: {{ getStatValue(currentPokemons[1], selectedStatPlayer2[currentPokemons[1].id]) }}
              </p>
            </div>
          </div>
        </div>
      </div>


      <div class="buttons">
        <button @click="determineWinner" class="next-button">Ver resultado de la ronda</button>
        <button @click="nextRound" class="next-button" v-if="winner">Siguiente Ronda</button>
        <button @click="reset" class="reset-button">Volver a seleccionar rondas</button>
      </div>


      <div v-if="winner" class="winner">
        <span v-if="winner.name === 'Empate'">¡Empate!</span>
        <span v-else>¡Ganador de la ronda: {{ capitalize(winner.name) }}!</span>
      </div>
    </div>


    <div v-if="battleEnded" class="final-winner">
      <h1>¡Ganador de la batalla: Jugador {{ finalWinner }}!</h1>
      <button @click="reset" class="reset-button">Volver a iniciar</button>
    </div>
  </div>
</template>

<script>
import axios from 'axios';
import { ref, onMounted } from 'vue'; 

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


    const capitalize = (str) => {
      if (!str) return '';
      return str.charAt(0).toUpperCase() + str.slice(1);
    };


    const fetchAllPokemons = async () => {
      loading.value = true;
      try {
        const response = await axios.get('https://pokeapi.co/api/v2/pokemon?limit=150');
        const pokemons = response.data.results;
        allPokemons.value = await Promise.all(pokemons.map(async (pokemon) => {
          const pokemonDetails = await axios.get(pokemon.url);
          return {
            id: pokemonDetails.data.id,
            name: pokemonDetails.data.name,
            stats: pokemonDetails.data.stats.map(stat => ({
              name: stat.stat.name,
              base_stat: stat.base_stat
            })),
            sprites: {
              front_default: pokemonDetails.data.sprites.front_default
            }
          };
        }));
      } catch (error) {
        console.error('Error fetching pokemons:', error);
      } finally {
        loading.value = false;
      }
    };


    const selectRounds = (number) => {
      rounds.value = number;
      selectingPokemons.value = true;
      currentPlayer.value = 1;
      selectedPokemonsPlayer1.value = [];
      selectedPokemonsPlayer2.value = [];
      selectedStatPlayer1.value = {};
      selectedStatPlayer2.value = {};
    };


    const selectPokemon = (pokemon) => {
      if (currentPlayer.value === 1) {
        if (!isPokemonSelected(pokemon) && selectedPokemonsPlayer1.value.length < rounds.value) {
          selectedPokemonsPlayer1.value.push(pokemon);
        }
      }

    };


    const isPokemonSelected = (pokemon) => {
      return selectedPokemonsPlayer1.value.some(p => p.id === pokemon.id) ||
             selectedPokemonsPlayer2.value.some(p => p.id === pokemon.id);
    };


    const nextPlayer = () => {
      if (currentPlayer.value === 1) {

        if (selectedPokemonsPlayer1.value.length !== rounds.value) {
          alert(`Selecciona exactamente ${rounds.value} Pokémon para el Jugador 1.`);
          return;
        }
        currentPlayer.value = 2;


        selectedPokemonsPlayer2.value = [];
        let availablePokemons = allPokemons.value.filter(p => !selectedPokemonsPlayer1.value.some(sp => sp.id === p.id));

        for (let i = 0; i < rounds.value; i++) {
          if (availablePokemons.length === 0) break;
          const randomIndex = Math.floor(Math.random() * availablePokemons.length);
          const selectedPokemon = availablePokemons.splice(randomIndex, 1)[0];
          selectedPokemonsPlayer2.value.push(selectedPokemon);
          
          // Asignar la misma estadística que el Jugador 1
          const player1Pokemon = selectedPokemonsPlayer1.value[i];
          selectedStatPlayer2.value[selectedPokemon.id] = selectedStatPlayer1.value[player1Pokemon.id];
        }


        if (selectedPokemonsPlayer2.value.length < rounds.value) {
          alert('No hay suficientes Pokémon disponibles para el Jugador 2.');
          reset();
        }
      }
    };


    const handleStatChange = (pokemonId, player) => {
      if (player === 1) {
        // Verificar si todas las estadísticas del Jugador 1 están seleccionadas
        if (areAllStatsSelected(selectedStatPlayer1.value)) {
          nextPlayer();
        }
      }
      // No se requiere lógica para el Jugador 2
    };


    const startBattle = () => {
   
      if (!areAllStatsSelected(selectedStatPlayer1.value) || !areAllStatsSelected(selectedStatPlayer2.value)) {
        alert('Asegúrate de haber seleccionado una estadística para cada Pokémon de ambos jugadores.');
        return;
      }


      currentRound.value = 0;
      currentPokemons.value = [
        selectedPokemonsPlayer1.value[currentRound.value],
        selectedPokemonsPlayer2.value[currentRound.value]
      ];
      battleStarted.value = true;
      selectingPokemons.value = false;
    };


    const determineWinner = () => {
      const player1Pokemon = currentPokemons.value[0];
      const player2Pokemon = currentPokemons.value[1];
      const statToComparePlayer1 = selectedStatPlayer1.value[player1Pokemon.id];
      const statToComparePlayer2 = selectedStatPlayer2.value[player2Pokemon.id];

      const player1StatValue = getStatValue(player1Pokemon, statToComparePlayer1);
      const player2StatValue = getStatValue(player2Pokemon, statToComparePlayer2);

      if (player1StatValue > player2StatValue) {
        winner.value = { name: player1Pokemon.name, player: 1 };
      } else if (player2StatValue > player1StatValue) {
        winner.value = { name: player2Pokemon.name, player: 2 };
      } else {
        winner.value = { name: 'Empate' };
      }
    };


    const getStatValue = (pokemon, statName) => {
      const stat = pokemon.stats.find(s => s.name === statName);
      return stat ? stat.base_stat : 0;
    };


    const nextRound = () => {
      if (currentRound.value < rounds.value - 1) {
        currentRound.value++;
        currentPokemons.value = [
          selectedPokemonsPlayer1.value[currentRound.value],
          selectedPokemonsPlayer2.value[currentRound.value]
        ];
        winner.value = null; 
      } else {

        battleEnded.value = true;
        const totalWinsPlayer1 = selectedPokemonsPlayer1.value.reduce((acc, pokemon, index) => {
          const currentWinner = determineRoundWinner(pokemon, selectedPokemonsPlayer2.value[index]);
          return acc + (currentWinner === 1 ? 1 : 0);
        }, 0);

        const totalWinsPlayer2 = selectedPokemonsPlayer2.value.reduce((acc, pokemon, index) => {
          const currentWinner = determineRoundWinner(selectedPokemonsPlayer1.value[index], pokemon);
          return acc + (currentWinner === 2 ? 1 : 0);
        }, 0);

        if (totalWinsPlayer1 > totalWinsPlayer2) {
          finalWinner.value = 1;
        } else if (totalWinsPlayer2 > totalWinsPlayer1) {
          finalWinner.value = 2;
        } else {
          finalWinner.value = 'Empate';
        }
      }
    };

    const determineRoundWinner = (player1Pokemon, player2Pokemon) => {
      const statToComparePlayer1 = selectedStatPlayer1.value[player1Pokemon.id];
      const statToComparePlayer2 = selectedStatPlayer2.value[player2Pokemon.id];
      const player1StatValue = getStatValue(player1Pokemon, statToComparePlayer1);
      const player2StatValue = getStatValue(player2Pokemon, statToComparePlayer2);

      if (player1StatValue > player2StatValue) {
        return 1;
      } else if (player2StatValue > player1StatValue) {
        return 2;
      } else {
        return 0; 
      }
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
      battleStarted.value = false;
      battleEnded.value = false;
      finalWinner.value = null;
      selectedStatPlayer1.value = {};
      selectedStatPlayer2.value = {};
    };


    const areAllStatsSelected = (stats) => {
      return (
        Object.values(stats).length === rounds.value &&
        Object.values(stats).every((stat) => stat !== undefined && stat !== "")
      );
    };

    const getPokemonTypeColor = (pokemon) => {
      if (pokemon.stats.length > 0) {
        const statPlayer1 = selectedStatPlayer1.value[pokemon.id];
        const statPlayer2 = selectedStatPlayer2.value[pokemon.id];
        // Priorizar la estadística del jugador que selecciona actualmente
        const stat = currentPlayer.value === 1 ? statPlayer1 : statPlayer2;
        switch (stat) {
          case 'hp':
            return '#FF0000'; 
          case 'attack':
            return '#FFA500'; 
          case 'defense':
            return '#008000'; 
          case 'speed':
            return '#0000FF'; 
          case 'special-attack':
            return '#FFD704'; // Corregido: Eliminado doble #
          case 'special-defense':
            return '#ff4500'; 
          default:
            return '#A8A878'; 
        }
      }
      return '#A8A878'; 
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
      isPokemonSelected,
      nextPlayer,
      startBattle,
      determineWinner,
      getStatValue,
      nextRound,
      reset,
      areAllStatsSelected,
      getPokemonTypeColor,
      capitalize,
      handleStatChange // Asegúrate de exportar esta función
    };
  },
};
</script>



<style scoped>

.selection, .pokemon-selection, .battles, .final-winner {
  backdrop-filter: none; 
  padding: 20px;
  margin-bottom: 20px;
  border: 2px solid white;
  border-radius: 10px;
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
  background-repeat: no-repeat;
  width: 100%;
  background-size: cover;
  position: relative; 
  z-index: 2; 
  font-size: 1.5rem; 
  color: #FFD700; 
  font-family: fantasy ;
  text-shadow: 2px 2px 5px rgba(0, 0, 0, 0.4); 
  transition: transform 0.3s ease; 
  text-align: center; 
  animation: bounceTitle 2s ease infinite; 
}

.rounds button, .next-button, .start-button, .reset-button {
  background: radial-gradient(circle, rgba(197,26,255,1) 0%, rgba(190,0,0,0.8757878151260504) 100%);
  color: rgb(0, 0, 0); 
  border: none;
  border-radius: 5px;
  padding: 15px 30px;
  font-size: 20px; 
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  margin: 10px;
}

.rounds button:hover, .next-button:hover, .start-button:hover, .reset-button:hover {
  background-color: rgba(0, 123, 255, 1); 
}

.pokemon-container {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  flex: 2;
}

.pokemon-card {
  margin: 20px;
  padding: 20px;
  border-radius: 10px;
  width: 180px;
  cursor: pointer;
  text-align: center;
  transition: transform 0.2s;
}

.pokemon-card:hover {
  transform: scale(1.05);
}

.pokemon-card.selected {
  transform: scale(1.1); 
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.5); 
  border: 2px solid #FFD700; 
}

.battles, .final-winner {
  background-image: url('https://i.pinimg.com/originals/89/a1/4d/89a14d3b81546e2fe51cbf4d0d6b8b60.gif');
  background-size: cover;
  background-position: center;
  padding: 20px;
  border-radius: 10px;
}

.battle-background {
  backdrop-filter: blur(10px);
  border-radius: 10px;
  padding: 40px;
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
  padding: 20px;
  border-radius: 8px;
  margin-top: 10px;
  font-size: 16px;
}

.buttons {
  margin-top: 20px;
  display: flex;
  justify-content: center;
  gap: 20px;
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

.loading {
  font-size: 24px;
  color: #555;
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
  margin-bottom: 10px;
}

.pokemon-image {
  width: 100px;
  height: 100px;
  border-radius: 50%;
}

.imagentitulo {
  background-color: transparent; 
  display: inline-block;
  margin: 0;
  padding: 0; 
}


.selection-container {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
}

.selected-pokemons {
  flex: 1;
  padding: 10px;
  text-align: left;
}

.selected-pokemons h2 {
  font-size: 24px;
  margin-bottom: 10px;
}

.selected-pokemons ul {
  list-style: none;
  padding: 0;
}

.selected-pokemons li {
  margin-bottom: 5px;
}


@media (max-width: 768px) {
  .selection-container {
    flex-direction: column;
    align-items: center;
  }

  .selected-pokemons {
    width: 100%;
    text-align: center;
    margin-bottom: 20px;
  }

  .pokemon-container {
    width: 100%;
    justify-content: center;
  }
}

button, input, select {
  overflow: visible;
  text-transform: none;
  width: 70%;
}

</style>
