<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  _id: string
  _rev?: string
  doc: {
    post_name: string
    post_content: string
    attributes: {
      creation_date: string
      modified: boolean
    }
  }
}

export default {
  data() {
    return {
      step: 0,
      total: 0,
      postsData: [] as Post[],
      document: null as Post | null,
      storage: null as PouchDB.Database | null,
      remoteDB: null as PouchDB.Database | null
    }
  },

  mounted() {
    this.initDatabase()
    this.initRemoteDatabase()
    this.updateLocalDatabase()
    this.fetchData()
  },

  methods: {
    async updateData(document: Post) {
      this.log('Call updateData')
      const db = this.storage

      // Vérifier si le stockage est bien défini
      if (!db) {
        this.log("Le stockage n'est pas défini.")
        return
      }

      try {
        // Récupérer le document existant pour obtenir son _rev (version)
        const existingDoc = await db.get(document._id)
        document._rev = existingDoc._rev // Assigner _rev pour la mise à jour
        // Mettre à jour le document
        await db.put(document)
        this.log('updateData success')
      } catch (error) {
        this.log('updateData error', error)
      }
    },

    async deleteData(id: string) {
      this.log('Call deleteData')
      const db = this.storage

      // Vérifier si le stockage est bien défini
      if (!db) {
        this.log("Le stockage n'est pas défini.")
        return
      }

      try {
        // Récupérer le document existant pour obtenir son _rev
        const existingDoc = await db.get(id)
        await db.remove(existingDoc._id, existingDoc._rev)
        this.log('deleteData success')
      } catch (error) {
        this.log('deleteData error', error)
      }
    },

    fetchData() {
      this.log('Call fetchData')
      const db = ref(this.storage).value
      const self = this
      if (db) {
        db.allDocs({
          include_docs: true,
          attachments: true
        })
          .then(function (result: any) {
            self.log('fetchData success', result)
            self.postsData = result.rows
          })
          .catch(function (error: any) {
            self.log('fetchData error', error)
          })
      }
    },

    fetchDistantData() {
      this.log('Call fetchData')
      const db = ref(this.remoteDB).value
      const self = this
      if (db) {
        db.allDocs({
          include_docs: true,
          attachments: true
        })
          .then(function (result: any) {
            self.log('fetchData success', result)
            self.postsData = result.rows
          })
          .catch(function (error: any) {
            self.log('fetchData error', error)
          })
      }
    },

    createData(document: Post) {
      this.log('Call createData', document)
      const db = ref(this.storage).value
      try {
        db?.post(document)
        this.log('createData success')
      } catch (error) {
        this.log('createData error', error)
        //throw new Error('createData error')
      }
    },

    initDatabase() {
      this.log('Call initDatabase')
      const db = new PouchDB('http://admin:admin@localhost:5984/database')
      if (db) {
        this.log('Connected to collection ', db.name)
      } else {
        this.log('Something went wrong')
      }
      this.storage = db
    },

    initRemoteDatabase() {
      this.log('Call initDatabase')
      const remoteDB = new PouchDB('http://admin:admin@localhost:5984/post')
      if (remoteDB) {
        this.log('Connected to collection ', remoteDB.name)
      } else {
        this.log('Something went wrong')
      }
      this.remoteDB = remoteDB
    },

    log(...o: any) {
      this.step++
      console.log('Step', this.step, ':', ...o)
    },

    updateLocalDatabase() {
      const db = ref(this.storage).value
      if (db) {
        db.replicate.from
          .bind(this)(this.remoteDB!)
          .on('complete', () => {
            console.log('on replicate complete')
            this.fetchData()
          })
          .on('error', function (error) {
            console.log('error', error)
          })
      }
    },

    updateDistantDatabase() {
      const remotDb = ref(this.remoteDB).value
      if (remotDb) {
        remotDb.replicate.from
          .bind(this)(this.storage!)
          .on('complete', () => {
            console.log('on replicate complete')
            this.fetchData()
          })
          .on('error', function (error: any) {
            console.log('error', error)
          })
      }
    },

    watchRemoteDatabase() {
      this.fetchDistantData()
    }
  }
}
</script>

<template>
  <div>
    <h1>Nombre de post: {{ postsData.length }}</h1>
    <ul>
      <li v-for="post in postsData" :key="post._id">
        <div class="ucfirst">{{ post.doc.post_name }} - {{ post.doc.post_content }}</div>
      </li>
    </ul>
  </div>
</template>
