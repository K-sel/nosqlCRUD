<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchFind from 'pouchdb-find'

PouchDB.plugin(PouchFind)

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
    media?: { name: string; type: string; data: string }[]
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
      localURL: 'K-sel',
      remoteURL: 'http://admin:admin@localhost:5984/database',
      localDB: null as PouchDB.Database | null,
      remoteDB: null as PouchDB.Database | null,
      postName: '',
      postContent: '',
      attributes: {
        creation_date: '',
        modified: false,
        modified_date: ''
      },
      mediaFiles: [] as File[],
      showAttachmentFields: {} as Record<string, boolean>
    }
  },

  mounted() {
    this.initLocalDatabase()
    this.initRemoteDatabase()
    this.setupReplication()
    this.watchRemoteDatabase()
    this.fetchData()
  },

  methods: {
    toggleAttachmentFields(postId: string) {
      if (!this.showAttachmentFields[postId]) {
        this.showAttachmentFields[postId] = true // Ajouter la clé si elle n'existe pas
      } else {
        this.showAttachmentFields[postId] = !this.showAttachmentFields[postId] // Inverser la valeur
      }
    },

    generateDemoData() {
      const demoPosts = Array.from({ length: 20 }, (_, i) => ({
        id: `${Date.now()}_${i}`,
        doc: {
          post_name: `Demo Post ${i}`,
          post_content: `This is the content for demo post ${i}.`,
          attributes: {
            creation_date: new Date().toISOString(),
            modified: false
          }
        }
      }))
      demoPosts.forEach((post) => this.createData(post))
    },

    createIndex() {
      const localDB = this.localDB
      if (localDB) {
        localDB.createIndex({
          index: { fields: ['doc.post_name'] }
        })
          .then(() => {
            console.log('Index on post_name created successfully')
          })
          .catch((err) => {
            console.error('Error creating index:', err)
          })
      }
    },

    async addMediaToDocument(postId: string, files: File[]) {
      const localDB = this.localDB
      if (!localDB) return

      try {
        const doc = await localDB.get(postId)
        doc.media = doc.media || []
        for (const file of files) {
          const reader = new FileReader()
          reader.onload = async (e) => {
            doc.media.push({
              name: file.name,
              type: file.type,
              data: e.target?.result as string
            })
            await localDB.put(doc)
            console.log(`Media added to document ${postId}`)
          }
          reader.readAsDataURL(file)
        }
      } catch (err) {
        console.error('Error adding media:', err)
      }
    },

    async removeMediaFromDocument(postId: string, mediaName: string) {
      const localDB = this.localDB
      if (!localDB) return

      try {
        const doc = await localDB.get(postId)
        doc.media = doc.media?.filter((m) => m.name !== mediaName)
        await localDB.put(doc)
        console.log(`Media removed from document ${postId}`)
      } catch (err) {
        console.error('Error removing media:', err)
      }
    },

    handleMediaUpload(event: Event) {
      const files = (event.target as HTMLInputElement).files
      if (files) this.mediaFiles = Array.from(files)
    },

    handleMediaSubmit(postId: string) {
      if (this.mediaFiles.length > 0) {
        this.addMediaToDocument(postId, this.mediaFiles)
        this.mediaFiles = []
      }
    },

    createData(document: Post) {
      this.log('Call createData', document)
      const localDB = ref(this.localDB).value
      try {
        localDB?.post(document)
        this.log('createData success')
      } catch (error) {
        this.log('createData error', error)
      }
    },

    async searchPosts() {
      this.log('Début searchPosts avec query:', this.searchQuery)

      if (!this.localDB || !this.searchQuery) {
        this.fetchData()
        return
      }
      const queryRegex = `(?i)^${this.searchQuery}` // Expression régulière insensible à la casse

      try {
        this.log("Vérification de l'index...")
        this.log('Exécution de la recherche...')
        const result = await this.localDB.find({
          selector: {
            $or: [
              {
                'doc.post_name': {
                  $regex: queryRegex
                }
              }
            ]
          }
        })

        this.log('Résultat brut de find:', result)
        this.postsData = result.docs
      } catch (error) {
        console.error('Erreur pendant la recherche:', error)
        this.fetchData()
      }
    },

    handleSubmit() {
      const generateId = (): string => {
        return Math.random().toString(36).substring(2) + Date.now().toString(36)
      }

      const newPost: Post = {
        doc: {
          id: generateId(),
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
      const localDB = this.localDB

      if (!localDB) {
        this.log("Le stockage n'est pas défini.")
        return
      }

      try {
        const existingDoc = await localDB.get(document.id)
        document.doc._rev = existingDoc._rev
        await localDB.put(document.doc)
        this.log('updateData success')
        this.fetchData()
      } catch (error) {
        this.log('updateData error', error)
      }
    },

    modifyDocument(document: Post) {
      this.log('Call modifyDocument', document)

      // Demander les nouvelles valeurs
      const newName = prompt(
        "Modifier le nom du post (laissez vide pour conserver l'ancien nom) :",
        document.doc.doc.post_name
      )
      const newContent = prompt(
        "Modifier le contenu du post (laissez vide pour conserver l'ancien contenu) :",
        document.doc.doc.post_content
      )

      // Vérifier si aucune modification n'a été apportée
      if (newName === '' && newContent === '') {
        alert('Modification annulée, aucune donnée saisie.')
        return
      }

      // Mettre à jour uniquement si des changements ont été effectués
      const hasChanged =
        (newName && newName !== document.doc.doc.post_name) ||
        (newContent && newContent !== document.doc.doc.post_content)
      if (hasChanged) {
        if (newName) {
          document.doc.doc.post_name = newName
        }
        if (newContent) {
          document.doc.doc.post_content = newContent
        }

        // Marquer le document comme modifié
        document.doc.doc.attributes.modified = true
        document.doc.doc.attributes.modified_date = new Date().toISOString()

        // Appeler la mise à jour des données
        this.updateData(document)
      } else {
        alert('Modification annulée, aucune donnée modifiée.')
      }
    },

    async deleteData(id: string) {
      this.log('Call deleteData', id)
      const localDB = this.localDB

      if (!localDB) {
        this.log("Le stockage n'est pas défini.")
        return
      }

      try {
        if (!id || typeof id !== 'string') {
          throw new Error(`ID invalide : ${id}`)
        }

        this.log('Fetching document for deletion:', id)

        const existingDoc = await localDB.get(id)

        if (!existingDoc) {
          throw new Error(`Document avec ID ${id} introuvable.`)
        }

        await localDB.remove(existingDoc)
        this.log('deleteData success')

        this.postsData = this.postsData.filter((post) => post.id !== id)
      } catch (error) {
        this.log('deleteData error', error)
      }
    },

    fetchData() {
      this.log('Call fetchData')
      const localDB = ref(this.localDB).value

      if (localDB) {
        localDB.allDocs({
          include_docs: true,
          attachments: true
        })
          .then((result: any) => {
            this.log('Structure complète allDocs:', result)
            const filteredRows = result.rows.filter((row) => !row.id.startsWith('_design/'))
            this.log('fetchData success', filteredRows)
            this.postsData = filteredRows
          })
          .catch((error: any) => {
            this.log('fetchData error', error)
          })
      }
    },

    fetchDistantData() {
      this.log('Call fetchDistantData')
      const remoteDB = ref(this.remoteDB).value

      if (remoteDB) {
        remoteDB.allDocs({
          include_docs: true,
          attachments: true
        })
          .then((result: any) => {
            this.log('fetchDistantData result:', result)
            const filteredRows = result.rows.filter((row) => !row.id.startsWith('_design/'))
            this.log('fetchDistantData success', filteredRows)
            this.postsData = filteredRows
          })
          .catch((error: any) => {
            this.log('fetchDistantData error', error)
          })
      }
    },

    initLocalDatabase() {
      this.log('Call initLocalDatabase')
      const localDB = new PouchDB(this.localURL)
      if (localDB) {
        this.log('Connected to collection ', localDB.name)
        localDB
          .createIndex({
            index: {
              fields: ['post_name']
            }
          })
          .then(() => {
            this.log('Index créé avec succès')
          })
          .catch((error) => {
            console.error('Erreur création index:', error)
          })
      } else {
        this.log('Something went wrong')
      }
      this.localDB = localDB
    },

    initRemoteDatabase() {
      this.log('Call initRemoteDatabase')
      const remoteDB = new PouchDB(this.remoteURL)
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
      const localDB = ref(this.localDB).value
      if (localDB) {
        localDB.replicate.from
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
          .bind(this)(this.localDB!)
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
      const localDB = this.localDB
      if (!localDB) return

      try {
        const doc = await localDB.get(postId)
        const result = await localDB.putAttachment(postId, file.name, doc._rev, file, file.type)
        console.log('Attachment added', result)
      } catch (error) {
        console.error('Attachment error', error)
      }
    },

    setupReplication() {
      if (!this.localDB || !this.remoteDB) return
      this.log('setupReplication')
      this.remoteDB.replicate
        .to(this.localDB, {
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
        .on('complete', (complete) => {
          console.log('on Complete', complete)
        })

      this.localDB.replicate
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
      <div class="left-column">
        <div class="search-section">
          <input v-model="searchQuery" @input="searchPosts" type="text" class="search-input"
            placeholder="Rechercher des posts..." />
        </div>

        <div class="posts-list" v-if="postsData.length > 0">
          <h3>Posts existants</h3>
          <div v-for="post in postsData" :key="post.id" class="post-item">
            <div class="post-content">
              <h4>{{ post?.doc?.post_name || post?.doc?.doc?.post_name || 'Sans titre' }}</h4>
              <p>
                {{ post?.doc?.post_content || post?.doc?.doc?.post_content || 'Pas de contenu' }}
              </p>
              <div v-if="post.doc.media">
                <h5>Médias associés :</h5>
                <ul>
                  <li v-for="media in post.doc.media" :key="media.name">
                    {{ media.name }}
                    <button @click="removeMediaFromDocument(post.id, media.name)" class="delete-button">
                      Supprimer
                    </button>
                  </li>
                </ul>
              </div>
            </div>
            <div class="post-actions">
              <button @click="deleteData(post.id)" class="delete-button">Supprimer</button>
              <button @click="modifyDocument(post)" class="modify-button">Modifier</button>
              <button @click="toggleAttachmentFields(post.id)" class="attachment-button">
                Attachments
              </button>
            </div>
            <div v-if="showAttachmentFields[post.id]" class="attachment-fields">
              <input type="file" multiple @change="handleMediaUpload" class="file-input" />
              <button @click="handleMediaSubmit(post.id)" class="add-media-button">
                Ajouter Médias
              </button>
            </div>
          </div>
        </div>
        <div v-else class="no-posts">Aucun post disponible</div>
      </div>

      <div class="right-column">
        <div class="create-post-form">
          <h2>Créer un nouveau post</h2>
          <form @submit.prevent="handleSubmit" class="form">
            <div class="form-group">
              <label for="post-name">Nom du post</label>
              <input id="post-name" v-model="postName" type="text" required class="form-input"
                placeholder="Entrez le nom du post" />
            </div>

            <div class="form-group">
              <label for="post-content">Contenu du post</label>
              <textarea id="post-content" v-model="postContent" required class="form-textarea"
                placeholder="Entrez le contenu du post" rows="4"></textarea>
            </div>
            <button type="submit" class="submit-button">Créer post</button>
            <button @click="generateDemoData" class="demo-button">Générer des données démo</button>
          </form>
        </div>
      </div>
    </div>
  </div>
</template>
