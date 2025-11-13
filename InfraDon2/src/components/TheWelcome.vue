<script setup lang="ts">
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

const counter = ref(0)
const increment = () => {
  counter.value++
}

const storage = ref<PouchDB.Database | undefined>()
const postsData = ref<Post[]>([])
const isOffline = ref(false)

const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  
  const localDB = new PouchDB('Posts')
  storage.value = localDB
  console.log('Base locale :', localDB?.name)

  if (!isOffline.value) {
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
}

const fetchData = () => {
  storage.value?.allDocs({
    include_docs: true,
    attachments: true,
  })
  .then((result: any) => {
    postsData.value = result.rows.map((row: any) => row.doc)
    console.log(postsData.value)
  })
  .catch((err: any) => {
    console.log('Erreur pour récupérer des données', err)
  })
}

const createDoc = (newDoc: any) => {
  storage.value?.post(newDoc)
    .then((response: any) => {
      console.log(response)
      fetchData()
    })
    .catch((err: any) => {
      console.log('Erreur lors de la création du document :(', err)
    })
}

const updateDoc = (doc: any) => {
  const newDoc = { ...doc }
  newDoc.ville = 'Update ' + new Date().toISOString()
  storage.value?.put(newDoc)
    .then((response: any) => {
      console.log(response)
      fetchData()
    })
    .catch((err: any) => {
      console.log('Erreur lors de la mise à jour du document', err)
    })
}

const deleteDoc = (doc: any) => {
  storage.value?.remove(doc)
    .then((response: any) => {
      console.log(response)
      fetchData()
    })
    .catch((err: any) => {
      console.log('Erreur lors de la suppression du document', err)
    })
}

// Fonction pour basculer entre le mode offline et online
const toggleOfflineMode = () => {
  isOffline.value = !isOffline.value
  if (isOffline.value) {
    console.log("Mode offline activé.")
    storage.value?.replicate.cancel() 
  } else {
    console.log("Mode online activé.")
    initDatabase()  
  }
}

const MAJserveur = async () => {
  const remoteDB = new PouchDB('http://nuno:Nunofaria17@localhost:5984/infradonn2_chat');
  
  try {
    await storage.value?.replicate.from(remoteDB);
    console.log('Données récupérées du serveur');

    await storage.value?.replicate.to(remoteDB);
    console.log('Données envoyées au serveur');

    fetchData();
  } catch (err) {
    console.error('Erreur de synchronisation :(', err);
  }
};

//13.11 creation index
const createIndex = () => {
  storage.value?.createIndex({
    index: { fields: ['nom'] }
  }).then((result) => {
    console.log('Index créé sur le champ "nom"', result);
  }).catch((err) => {
    console.error('Erreur lors de la création de l’index', err);
  })
}

// Recherche d'un document par nom
const searchByName = (searchTerm: string) => {
  storage.value?.find({
    selector: { nom: { $eq: searchTerm } },
    fields: ['_id', 'nom', 'age', 'ville', 'sport']
  }).then((result: any) => {
    console.log('Résultats de la recherche:', result.docs)
    postsData.value = result.docs
  }).catch((err) => {
    console.error('Erreur lors de la recherche', err)
  })
}

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()  
  fetchData()
  createIndex()  
})
</script>

<template>
  <h1>Posts - Pratique InfraDon2</h1>
  <p>Counter: {{ counter }}</p>
  <button @click="increment">+1</button>
  
  <div>
    <button @click="toggleOfflineMode">
      {{ isOffline ? 'Passer en mode Online' : 'Passer en mode Offline' }}
    </button>
  </div>
  
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

  <div>
    <input type="text" v-model="searchTerm" placeholder="Rechercher par nom" />
    <button @click="searchByName(searchTerm)">
      Rechercher
    </button>
  </div>
</template>
