<template>
  <div v-bind:class="{ 'active': race.selected }" v-on:click=raceSelected>
    Étape {{ race.stage }} : {{ race.data.name }}
  </div>
</template>

<script>
export default {
  name: 'Race',

  props: {
    race: Object
  },

  methods: {
    raceSelected: function (e) {
      this.$dispatch('race-selected', this.race)
    }
  }
}
</script>

<style scoped>
div{
  color: #000;
  font-family: 'Lato', sans-serif;
  font-weight: 300;
}
.active {
  background: #f5da3f;
  font-weight: 700;
}

div:hover{
  cursor: pointer;
  background: rgba(255, 255, 0, 0.6);
}
</style>
