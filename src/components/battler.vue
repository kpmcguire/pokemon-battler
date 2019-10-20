<template lang="pug">
  form(v-on:submit.prevent)
    .flex.items-center.tombstone.battleground(v-if="!loaded") 
      each r in [...Array(2).keys()]
        div(class="pokewrap w-2/5 p-5") 
          div.text-center(class="md:p-10")
            .avatar-wrap
              .avatar
            h2.font-bold.text-lg.mt-2(class="md:mt-4")
              span.placeholder(class="w-12 sm:w-64") &nbsp;              
      div.vs.text-center.text-4xl.text-red-600(class="w-1/5") 
        span.placeholder(class="w-12") &nbsp;

    .flex.items-center.battleground(v-bind:class="{active: isActive}" v-if="loaded")
      div(v-for="(p, index) in pokemon" :key="p.id" class="pokewrap w-2/5 p-5") 
        div.text-center
          //- ul.list-none(v-if="p.abilities")
          //-   li(v-for="(x) in p.abilities") {{x.ability.name}} 

          label
            .avatar-wrap
              img.block.w-full.avatar(v-bind:src="p.sprites.front_default" v-bind:id="'combatant-'+index" class="")
              .selected-status(v-if="selectedBattler")
                .text-green-400(v-if="selectedBattler === p.name")
                  include ../assets/images/check-circle-solid.svg
                .text-red-400(v-else)
                  include ../assets/images/times-circle-solid.svg              
            span(class="inline-block fake-button bg-red-600 hover:bg-red-700 active:bg-red-900 py-2 px-3 rounded text-white font-bold text-xs md:text-xl")
              input(type="radio" name="selectedBattler" @change="allowFightCheck" v-bind:value="p.name" v-model="selectedBattler")
              span(class="hidden md:inline") I think 
              span.capitalize {{p.name}} 
              span(class="hidden md:inline") will win
      div.vs.text-center(class="w-1/5") 
        span.bg-red-600.text-white.rounded-full.font-bold.border-4.border-white(class="sm:text-2xl md:3xl md:px-4 md:py-5 px-2 py-3") Vs.   
    .flex.flex-col.bg-blue-800(class="md:flex-row")
        div(class="w-full md:w-1/3 ")
          .text-white(class="text-3xl md:text-5xl p-4 py-0 md:px-8 md:pt-4 md:pb-0") {{cash | toCurrency}}      
          div(class="p-4 py-2 md:p-8 md:py-2")
            .flex.flex-wrap.w-full
              label.w-full(for='wager')
                .block.mb-2.text-white Wager
            .flex.items-start
              input#wager(type="number" v-model="wager" ref="wagerInput" v-on:keydown="checkWager" class="flex-auto w-12shadow appearance-none border rounded py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline") 
              button(class="flex-none bg-green-500 hover:bg-green-700 text-white font-bold py-2 px-4 ml-4 rounded" v-on:click="betMax") 
                  span(class="hidden md:inline") Bet 
                  span Max     
          div(class="p-4 md:px-8 md:py-4")
            ul.text-yellow-400(v-if="errors.length") 
              li(v-for="error in errors") {{error}}
            ul.text-yellow-400(v-else-if="!selectedBattler") 
              li Pick a 'mon
            ul.text-green-300(v-else) 
              li Let's go!
            button.w-full(value="fight" :disabled="!isValid" v-bind:class="{disabled: !isValid}" v-on:click="announceWinner" class="bg-orange-500 hover:bg-orange-700 text-white font-bold py-2 px-6 my-4 text-5xl rounded") Fight!   
        .p-4(class="w-full md:w-2/3  md:p-10 order-first md:order-last")              
          .battle-output-wrapper
            #battle-output.output.text-left(v-html="output" ref="output" class="h-24 md:h-64")

</template>

<script>
import _ from 'lodash'
import axios from 'axios'
const pokeapi_url = 'https://pokeapi.co/api/v2/pokemon'
// const pokeapi_url = ''

let rand1 = 0
let rand2 = 0

export default {
  name: 'battler',
  props: {
    msg: String
  },

  data: function() {
    return {
      errors: [],
      pokemon: [],
      isActive: false,
      effectiveness: [`It's not very effective...`, `It's super effective!`],
      cash: 0,
      selectedBattler: '',
      winLossRecord: [],
      streak: 1, 
      wager: null,
      validWager: false,
      loaded: false,
      allowFight: false, 
      matchOver: true,
      output: ''
    }
  },
  created() {
    if (localStorage.getItem('cash') && ( Number(localStorage.getItem('cash')) > 0 )) {
      this.cash = Number(localStorage.getItem('cash'))
    } else {
      this.cash = 1000
    }
    this.loadPokemons()
  },
  methods: {
    getRandomInt(min, max) {
      min = Math.ceil(min)
      max = Math.floor(max)
      return Math.floor(Math.random() * (max - min + 1)) + min
    },
    announceWinner(){
      this.isActive = !this.isActive

      function abilityStatement(pokemon) {
        return `<span class="capitalize font-bold">${pokemon.name}</span> used <span class="capitalize font-bold">${pokemon.abilities[Math.floor(Math.random()*pokemon.abilities.length)].ability.name}</span>`
      }

      let effectivenessStatement = () => {
        return this.effectiveness[Math.floor(Math.random()*this.effectiveness.length)]
      }

      let setWinStatement = (winner, loser, selectedBattler) => {
        let statement = 
          `
            <span class="capitalize font-bold text-yellow-400">
              ${winner.name}
            </span> 
            <span>
              defeats
            </span>  
            <span class="capitalize font-bold text-yellow-400">
              ${loser.name}
            </span>                        
          `

          if (winner.name === selectedBattler) {
            statement += `<div class="">You win!</div> `
          } else {
            statement += `<div class="">You lose!</div> `
          }

          return statement
      }

      let statement1 = abilityStatement(this.pokemon[0])
      let statement2 = effectivenessStatement()
      let statement3 = abilityStatement(this.pokemon[1])
      let statement4 = effectivenessStatement()

      let winStatement = setWinStatement(this.sorted[0], this.sorted[1], this.selectedBattler)

      setTimeout(()=>{
        this.output += `${statement1}<br/>`

        var container = this.$refs.output;
        container.scrollTop = container.clientHeight;

      }, 150)

      setTimeout(()=>{

        this.output += `${statement2}<br/>`
        var container = this.$refs.output;
        container.scrollTop = container.clientHeight;

      }, 1150)      

      setTimeout(()=>{


        this.output += `${statement3}<br/>`
        var container = this.$refs.output;
        container.scrollTop = container.clientHeight;

      }, 1500)            

      setTimeout(()=>{

        this.output += `${statement4}<br/>`
        var container = this.$refs.output;
        container.scrollTop = container.clientHeight;

      }, 2500)            

   
      setTimeout(()=>{   
        this.matchOver = false

        this.output += `${winStatement}<br/>`

        if (this.sorted[0].name === this.selectedBattler) {

          if (_.last(this.winLossRecord) == true) {
            this.streak++
          } 

          this.cash += (this.wager * this.streak)
          this.winLossRecord.push(true)
          localStorage.setItem('cash', this.cash)

        } else {
          this.winLossRecord.push(false)
          this.streak = 1
          this.cash -= this.wager
          localStorage.setItem('cash', this.cash)
        }        
      }, 3000)

      setTimeout(()=>{   
        this.pokemon = []
        this.loadPokemons()
        this.isActive = false
        this.matchOver = true
      }, 5000)
    },
    loadPokemons() {
      this.loaded = false
      this.selectedBattler = ''
      this.validWager = false
      this.checkWager()
      const count = 807

      function getPokemon(id) {
        return axios.get(`${pokeapi_url}/${id}`)
      }

      rand1 = this.getRandomInt(1, count)
      rand2 = this.getRandomInt(1, count)

      axios.all([getPokemon(rand1), getPokemon(rand2)])
        .then(axios.spread((first, second) => {
          [first, second].forEach((pokemon) => {
            this.pokemon.push({
              abilities: pokemon.data.abilities,
              name: pokemon.data.name,
              sprites: pokemon.data.sprites,
              weight: pokemon.data.weight
            })
            this.loaded = true
          })     
        }))         
    }, 
    checkWager: _.debounce(function() {
      this.matchOver = true
      if (this.wager < this.cash && !isNaN(parseFloat(this.wager)) && isFinite(this.wager)) {
        this.validWager = true
        this.errors = []
        this.allowFightCheck()
        return true
      }

      this.errors = [];

      if (this.wager > this.cash && this.cash > 0) {
        this.errors.push('you don\'t have that much money')
        this.$refs.wagerInput.focus()
        this.validWager = false
      }

      if ((isNaN(parseFloat(this.wager)) && isFinite(this.wager))) {
        this.errors.push('enter a number')
        this.validWager = false
        this.$refs.wagerInput.focus()
      }
      
      this.allowFightCheck()
      // e.preventDefault()
    }, 500),
    allowFightCheck() {
      if (this.validWager && this.selectedBattler !== '') {
        this.allowFight = true
      } else {
        this.allowFight = false
      }
    },
    betMax() {
      if (this.cash > 0 ) {
        this.wager = this.cash
        this.validWager = true
        this.checkWager()
      }
    }
  },  
  computed: {
    sorted() {
      return _.sortBy(this.pokemon, 'weight').reverse();
    },
    isValid() {
      return this.allowFight
    }
  }, 
  filters: {
    toCurrency(value) {
      if (typeof value !== "number") {
          return value;
      }
      var formatter = new Intl.NumberFormat('en-US', {
          style: 'currency',
          currency: 'USD',
          minimumFractionDigits: 0
      });
      return formatter.format(value);
    } 
  }, 
  updated: function() {
    this.$nextTick(function () {
      var container = this.$refs.output;
      container.scrollTop = container.scrollHeight;
    })
  }
}

</script>
<style lang="scss">
  @import "@/assets/styles/customizations.scss"
</style>