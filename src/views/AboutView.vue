<script lang="ts">
import PouchDB from 'pouchdb'

export default {
  data() {
    return {
      datas: [],
      databaseReference: null as PouchDB.Database | null
    }
  },

  methods: {
    inc() {},

    initDatabase() {
      const db = new PouchDB('http://admin:admin@localhost:5984/post')
      if (db) {
        console.log("Connected to collection 'post'")
      } else {
        console.warn('Something went wrong')
      }
      this.databaseReference = db
    },

    fetchData() {
      // récupérer tous les documents de ma collection
    }
  },

  mounted() {
    console.log('hello')
  }
}
</script>

<template>
  <h1>InfraDon2</h1>
  <p>Counter: {{ datas }}</p>
  <button type="button" role="button" @click="inc">+1</button>
</template>
