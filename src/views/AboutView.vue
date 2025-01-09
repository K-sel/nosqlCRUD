<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  id: string
  doc: {
    _rev?: string
    _id?: string
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
      searchQuery: '',
      document: null as Post | null,
      storage: null as PouchDB.Database | null,
      remoteDB: null as PouchDB.Database | null
    }
  },

  mounted() {
    this.initDatabase()
    this.initRemoteDatabase()
    this.setupReplication()
    this.watchRemoteDatabase()
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
        const existingDoc = await db.get(document.id)
        document.doc._rev = existingDoc._rev // Assigner _rev pour la mise à jour
        // Mettre à jour le document
        await db.put(document)
        this.log('updateData success')
      } catch (error) {
        this.log('updateData error', error)
      }
    },

    async deleteData(id: string) {
      this.log('Call deleteData', id)
      const db = this.storage

      if (!db) {
        this.log("Le stockage n'est pas défini.")
        return
      }

      try {
        // Vérification de l'ID
        if (!id || typeof id !== 'string') {
          throw new Error(`ID invalide : ${id}`)
        }

        this.log('Fetching document for deletion:', id)

        // Récupérer le document existant
        const existingDoc = await db.get(id)

        if (!existingDoc) {
          throw new Error(`Document avec ID ${id} introuvable.`)
        }

        // Supprimer le document
        await db.remove(existingDoc)
        this.log('deleteData success')

        // Mettre à jour la liste des posts après suppression
        this.postsData = this.postsData.filter((post) => post.id !== id)
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
    async generateFakePost() {
      const fakePost: Post = {
        id: new Date().toISOString(),
        doc: {
          post_name: `Fake Post ${Math.random().toString(36).substring(7)}`,
          post_content: 'This is a randomly generated post.',
          attributes: {
            creation_date: new Date().toISOString(),
            modified: false
          }
        }
      }
      this.createData(fakePost)
    },

    async searchPosts() {
      const db = this.storage
      if (!db) return

      try {
        const result = await db.find({
          selector: { 'doc.post_name': { $regex: this.searchQuery } }
        })
        this.postsData = result.docs.map((doc: any) => ({
          id: doc._id,
          doc: doc
        })) as Post[]
      } catch (error) {
        console.error('Search error', error)
      }
    },

    async addAttachment(postId: string, file: File) {
      const db = this.storage
      if (!db) return

      try {
        const doc = await db.get(postId)
        const result = await db.putAttachment(postId, file.name, doc._rev, file, file.type)
        console.log('Attachment added', result)
      } catch (error) {
        console.error('Attachment error', error)
      }
    },

    setupReplication() {
      if (!this.storage || !this.remoteDB) return

      // Continuous replication from remote to local
      this.remoteDB.replicate
        .to(this.storage, {
          live: true,
          retry: true
        })
        .on('change', (info) => {
          console.log('Replication change', info)
          this.fetchData()
        })
        .on('error', (err) => {
          console.error('Replication error', err)
        })

      // Continuous replication from local to remote
      this.storage.replicate
        .to(this.remoteDB, {
          live: true,
          retry: true
        })
        .on('change', (info) => {
          console.log('Replication change', info)
        })
        .on('error', (err) => {
          console.error('Replication error', err)
        })
    },

    watchRemoteDatabase() {
      if (!this.remoteDB) return

      this.remoteDB
        .changes({
          since: 'now',
          live: true,
          include_docs: true
        })
        .on('change', (change) => {
          console.log('Remote database change detected', change)
          this.fetchData()
        })
        .on('error', (err) => {
          console.error('Error watching remote database', err)
        })
    }
  }
}
</script>

<template>
  <div>
    <h1>Gestion des Posts</h1>

    <!-- Recherche -->
    <input v-model="searchQuery" placeholder="Rechercher un post" />
    <button @click="searchPosts">Rechercher</button>

    <!-- Liste des posts -->
    <h2>Nombre de posts : {{ postsData.length }}</h2>
    <ul>
      <li v-for="post in postsData" :key="post.doc._id">
        <div>
          <strong>{{ post }} {{ post.doc.post_name }}</strong> - {{ post.doc.post_content }}
          <button @click="deleteData(post.doc._id as string)">Supprimer</button>
          <button @click="updateData(post)">Modifier</button>
          <input type="file" @change="(e) => addAttachment(post.id, (e.target as any).files[0])" />
        </div>
      </li>
    </ul>

    <!-- Ajout de posts -->
    <button @click="generateFakePost">Générer un post aléatoire</button>
  </div>
</template>
