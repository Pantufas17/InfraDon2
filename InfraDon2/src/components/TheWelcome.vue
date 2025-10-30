<script setup lang="ts">
const counter = ref(0)
const increment = () => {
  counter.value++
}

import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  nom: string
  age: number
  ville: string
  sport: string
  _id?: string
  _rev?: string
}

const storage = ref()

const postsData = ref<Post[]>([])

const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const db = new PouchDB('http://nuno:Nunofaria17@localhost:5984/infradonn2_chat')
  if (db) {
    console.log('Connecté à la collection : ' + db?.name)
    storage.value = db
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}

// recuperer des données
const fetchData = () => {
  storage.value
    .allDocs({
      include_docs: true,
      attachments: true,
    })
    .then((result: any) => {
      postsData.value = result.rows.map((row: any) => row.doc)
      console.log(postsData.value)
      // handle result
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const createDoc = (newDoc: any) => {
  storage.value
    .post(newDoc)
    .then((response: any) => {
      console.log(response)
      fetchData()
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const updateDoc = (doc: any) => {
  const newDoc = { ...doc }
  newDoc.ville = 'Update' + new Date().toISOString()
  storage.value
    .put(newDoc)
    .then((response: any) => {
      console.log(response)
      fetchData()
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const deleteDoc = (doc: any) => {
  storage.value
    .remove(doc)
    .then((response: any) => {
      console.log(response)
      fetchData()
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  fetchData()
})
</script>

<template>
  <h1>Posts ^^</h1>
  <p>Counter: {{ counter }}</p>
  <button @click="increment">+1</button>
  <p>PostDatas</p>
  <ul>
    <li v-for="post in postsData" :key="post._id">
      {{ post.nom }} - {{ post.age }} - {{ post.ville }}
      <button @click="updateDoc(post)">Modifier</button>
      <button @click="deleteDoc(post)">Supprimer</button>
    </li>
  </ul>
  <button @click="createDoc({ nom: 'Nuno', age: 22, ville: 'Lausanne', sport: 'foot' })">
    Créer
  </button>
</template>
