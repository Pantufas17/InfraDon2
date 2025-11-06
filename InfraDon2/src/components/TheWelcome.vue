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
  
  const localDB = new PouchDB('Posts')
  storage.value = localDB
  console.log('Base locale :', localDB?.name)

  localDB.replicate.from('http://nuno:Nunofaria17@localhost:5984/infradonn2_chat', {
    live: true, 
    retry: true, 
  }).then(() => {
    console.log('Réplication terminée')
    fetchData()  
  }).catch((err) => {
    console.log('Erreur, on peut pas faire réplication :(', err)
  })
}


const fetchData = () => {
  storage.value
    .allDocs({
      include_docs: true,
      attachments: true,
    })
    .then((result: any) => {
      postsData.value = result.rows.map((row: any) => row.doc)
      console.log(postsData.value)
    })
    .catch((err: any) => {
      console.log('Erreur pour recuperer des donnes', err)
    })
}


const createDoc = (newDoc: any) => {
  storage.value
    .post(newDoc)
    .then((response: any) => {
      console.log(response)
      fetchData()
    })
    .catch((err: any) => {
      console.log('Erreur quand on créer des docs :(', err)
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
    .catch((err: any) => {
      console.log('Erreur lors de la mise à jour du document', err)
    })
}

const deleteDoc = (doc: any) => {
  storage.value
    .remove(doc)
    .then((response: any) => {
      console.log(response)
      fetchData()
    })
    .catch((err: any) => {
      console.log('Erreur lors de la suppression du document', err)
    })
}

const MAJserveur = async () => {
  const remoteDB = new PouchDB('http://nuno:Nunofaria17@localhost:5984/infradonn2_chat');
  
  try {
    await storage.value.replicate.from(remoteDB);
    console.log('Données récupérées du serveur');

    await storage.value.replicate.to(remoteDB);
    console.log('Données envoyées au serveur');

    fetchData();
  } catch (err) {
    console.error('Erreur de synchronisation :(', err);
  }
};

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

  <button @click="MAJserveur">
    Mettre à jour le serveur. Envoyer les données.
  </button>
</template>
