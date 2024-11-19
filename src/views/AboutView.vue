<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'

/*
REVIEW Attention dans votre interface, car vous créez vous même un premier niveau "doc", donc une fois insérées, les informations seront dans 'doc.doc'
(le premier niveau 'doc' vient du couchDB allDocs et le second est le vôtre)
J'ai par ailleurs ajouté un "update_date" pour le debug

declare interface Post {
  _id: string
  _rev?: string
  doc: { // J'ai supprimé ce niveau donc 
    post_name: string
    post_content: string
    attributes: {
      creation_date: string
      modified: boolean
    }
  }
}

*/

declare interface Post {
  _id?: string
  _rev?: string
  post_name: string
  post_content: string
  attributes: {
    creation_date: string
    update_date: string
    modified: boolean
  }
}

const FAKE_POST: Post = {
  _id: '1',
  post_name: 'Post 1',
  post_content: 'Contenu du post 1',
  attributes: {
    creation_date: new Date().toISOString(),
    update_date: new Date().toISOString(),
    modified: false
  }
}

export default {
  data() {
    return {
      step: 0,
      total: 0,
      postsData: [] as Post[],
      document: null as Post | null,
      storage: null as PouchDB.Database | null
    }
  },

  /*
  REVIEW
  Lorsque la base de données est vide. Vous faites :
  - createData avec _id : 1 => La création se passe bien dans la base de données, mais on attend pas la fin l'insertion (cf method createData)
  - updateData avec _id : 1 => Ici, comme on a pas attendu la fin de l'insertion, on tente de récupérer le document avec l'_id 1, on trouve rien car le document n'est pas encore créé,
  donc, on ne peut pas mettre à jour

  Lorsque que l'on reload la page :
  - createData avec _id : 1 => La création failed ici car on a déjà un document avec l'_id 1
  Cependant, on affiche quand même 'createData success' car le log est exécuter sans attendre l'insertion et ensuite, on a la vraie erreur :
  {id: '1', error: 'conflict', reason: 'Document update conflict.', name: 'conflict', status: 409 }  
  - updateData avec _id = 1 => Ici, on trouve bien le document existant, la mise à jour se passe bien 

  
  */
  async mounted() {
    this.initDatabase()
    this.fetchData()
    /* REVIEW VERSION INITIAL
    //this.createData({ ...data })
    //this.updateDocument()

    /* REVIEW VERSION ASYNC à partir du moment où j'attends correctement la fin de la création du document (avec await/async),
    La mise à jour s'effectue correctement car le document est bien présent pour le mettre à jour
    */

    //await this.createDataAsync({ ...data })
    //this.updateDocument()
    /* REVIEW VERSIN PROMISE
     */
    this.createDataPromise({ ...FAKE_POST }, this.updateDocument, true)
    /* REVIEW
    L'inconvénient des async, c'est qu'il faut remonter la chaîne des calls pour s'assurer que tout attente correctement
    L'autre inconvénient, c'est que cela 'bloque' l'execution du code
    L'avantage, c'est que cela reste très lisible

    Les promises avec des callback c'est un peu plus clean 
    */
  },

  methods: {
    updateDocument() {
      this.updateData({
        ...FAKE_POST,
        post_name: 'Post 1 Modified',
        attributes: {
          ...FAKE_POST.attributes,
          modified: true,
          update_date: new Date().toISOString()
        }
      }, true)
    },

    async updateData(document: Post, refresh: boolean = false) {
      this.log('Call updateData')
      const db = ref(this.storage).value;

      // Vérifier si le stockage est bien défini
      if (!db) {
        this.log("Le stockage n'est pas défini.")
        return
      }

      if (!document._id) {
        this.log('Document _id est requis pour la mise à jour.')
        return
      }

      try {
        // Récupérer le document existant pour obtenir son _rev (version)
        const existingDoc = await db.get(document._id)
        document._rev = existingDoc._rev // Assigner _rev pour la mise à jour
        // Mettre à jour le document
        await db.put(document)
        this.log('updateData success')
        refresh && this.fetchData()
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
          .then((result: any) => {
            self.log('fetchData success', result)
            self.postsData = result.rows.map((row: any) => row.doc)
          })
          .catch((error: any) => self.log('fetchData error', error))
      }
    },

    createData(document: Post) {
      this.log('Call createData', document)
      const db = ref(this.storage).value
      try {
        // REVIEW ici, on attend pas la fin du 'db.post', donc on enchaîne directement avec le log
        // Donc on affiche TOUJOURS 'createData success'
        // Comme on attend pas le 'dp.post', on obtiens pas l'erreur et on arrive pas dans le catch
        db?.post(document)
        this.log('createData success')
      } catch (error) {
        this.log('createData error', error)
        //throw new Error('createData error')
      }
    },

    async createDataAsync(document: Post) {
      this.log('Call createDataAsync', document)
      const db = ref(this.storage).value
      try {
        await db?.post(document)
        this.log('createDataAsync success')
      } catch (error) {
        this.log('createDataAsync error', error)
      }
    },

    createDataPromise(document: Post, callback?: Function, refresh: boolean = false) {
      this.log('Call createDataPromise', document)
      const db = ref(this.storage).value
      // REVIEW Ici je bind(this) pour rester dans mon composant 
      // C'est une alternative à l'usage de 'self' qu'il y a par exemple dans votre fetchData
      db
        ?.post.bind(this)(document)
        .then(() => {
          this.log('createDataPromise success', refresh)
          callback?.call(this)
          refresh && this.fetchData()
        })
        .catch((error) => {
          this.log('createDataPromise error', error)
          callback?.call(this)
        })
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

    log(...o: any) {
      this.step++
      console.log('Step', this.step, ':', ...o)
    }
  }
}
</script>

<template>
  <div>
    <h1>Nombre de post: {{ postsData.length }}</h1>
    <ul>
      <li v-for="post in postsData" :key="post._id">
        {{ post.post_name
        }}<em style="font-size: x-small" v-if="post.attributes?.creation_date">
          - {{ post.attributes?.creation_date }} / {{ post.attributes?.update_date }}
        </em>
        Modified? {{ post.attributes?.modified }}
      </li>
    </ul>
    <button role="button" type="button" @click="deleteData('1')">Delete</button>
  </div>
</template>
