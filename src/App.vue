<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'
import 'pouchdb-find'

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
      modified_date?: string
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
      remoteDB: null as PouchDB.Database | null,
      postName: '',
      postContent: '',
      attributes: {
        creation_date: '',
        modified: false,
        modified_date: ''
      }
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
    createData(document: Post) {
      this.log('Call createData', document)
      const db = ref(this.storage).value
      try {
        db?.post(document)
        this.log('createData success')
      } catch (error) {
        this.log('createData error', error)
      }
    },

    // searchPosts() {
    //   this.storage
    //     ?.find({
    //       selector: {
    //         $or: [
    //           { post_name: { $regex: this.searchQuery, $options: 'i' } },
    //           { post_content: { $regex: this.searchQuery, $options: 'i' } }
    //         ]
    //       }
    //     })
    //     .then((result) => {
    //       this.postsData = result.docs
    //       this.log('Search result', result)
    //     })
    //     .catch((error) => {
    //       console.error('Error searching posts', error)
    //     })
    // },

    handleSubmit() {
      const generateId = (): string => {
        return Math.random().toString(36).substring(2) + Date.now().toString(36)
      }

      const newPost: Post = {
        id: generateId(),
        doc: {
          post_name: this.postName,
          post_content: this.postContent,
          attributes: {
            creation_date: new Date().toISOString(),
            modified: false
          }
        }
      }

      this.createData(newPost)
      this.postName = ''
      this.postContent = ''
      this.fetchData()
    },

    async updateData(document: Post) {
      this.log('Call updateData :', document)
      const db = this.storage

      if (!db) {
        this.log("Le stockage n'est pas défini.")
        return
      }

      try {
        const existingDoc = await db.get(document.id)
        document.doc._rev = existingDoc._rev
        await db.put(document.doc)
        this.log('updateData success')
        this.fetchData()
      } catch (error) {
        this.log('updateData error', error)
      }
    },
    modifyDocument(document: Post) {
      this.log('Call modifyDocument', document)
      const newName = prompt('Modifier le nom du post:', document.doc.post_name)
      const newContent = prompt('Modifier le contenu du post:', document.doc.post_content)
      if (newContent !== null) {
        document.doc.doc.post_name = newName
        document.doc.doc.post_content = newContent
        document.doc.doc.attributes.modified = true
        document.doc.doc.attributes.modified_date = new Date().toISOString()
        this.updateData(document)
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
        if (!id || typeof id !== 'string') {
          throw new Error(`ID invalide : ${id}`)
        }

        this.log('Fetching document for deletion:', id)

        const existingDoc = await db.get(id)

        if (!existingDoc) {
          throw new Error(`Document avec ID ${id} introuvable.`)
        }

        await db.remove(existingDoc)
        this.log('deleteData success')

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

    initDatabase() {
      this.log('Call initDatabase')
      const db = new PouchDB('http://admin:admin@localhost:5984/database')
      if (db) {
        this.log('Connected to collection ', db.name)
        // db.createIndex({
        //   index: { fields: ['post_name', 'post_content'] }
        // })
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
  <div class="app-container">
    <div class="content-wrapper">
      <!-- Colonne de gauche: Posts et recherche -->
      <div class="left-column">
        <div class="search-section">
          <input
            v-model="searchQuery"
            @input="searchPosts"
            type="text"
            class="search-input"
            placeholder="Rechercher des posts..."
          />
        </div>

        <div class="posts-list" v-if="postsData.length > 0">
          <h3>Posts existants</h3>
          <div v-for="post in postsData" :key="post.id" class="post-item">
            <div class="post-content">
              <h4>{{ post.doc.doc.post_name }}</h4>
              <p>{{ post.doc.doc.post_content }}</p>
            </div>
            <div class="post-actions">
              <button @click="deleteData(post.id)" class="delete-button">Supprimer</button>
              <button @click="modifyDocument(post)" class="modify-button">Modifier</button>
            </div>
          </div>
        </div>
        <div v-else class="no-posts">Aucun post disponible</div>
      </div>

      <!-- Colonne de droite: Formulaire -->
      <div class="right-column">
        <div class="create-post-form">
          <h2>Créer un nouveau post</h2>
          <form @submit.prevent="handleSubmit" class="form">
            <div class="form-group">
              <label for="post-name">Nom du post</label>
              <input
                id="post-name"
                v-model="postName"
                type="text"
                required
                class="form-input"
                placeholder="Entrez le nom du post"
              />
            </div>

            <div class="form-group">
              <label for="post-content">Contenu du post</label>
              <textarea
                id="post-content"
                v-model="postContent"
                required
                class="form-textarea"
                placeholder="Entrez le contenu du post"
                rows="4"
              ></textarea>
            </div>

            <button type="submit" class="submit-button">Créer post</button>
          </form>
        </div>
      </div>
    </div>
  </div>
</template>
