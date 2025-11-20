<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'
PouchDB.plugin(PouchDBFind)

// --- INTERFACES ---
declare interface Comment {
  id: string
  text: string
  date: string
}

declare interface Post {
  nom: string
  age: number
  ville: string
  sport: string
  likes: number           // Nouveau
  comments: Comment[]     // Nouveau
  _id?: string
  _rev?: string
}

// --- STATE ---
const counter = ref(0)
const storage = ref<PouchDB.Database | undefined>()
const postsData = ref<Post[]>([])
const isOffline = ref(false)
const searchTerm = ref('')
let replicationHandle: PouchDB.Replication.Sync<Post> | null = null
const COUCH_DB_URL = 'http://nuno:Nunofaria17@localhost:5984/infradonn2_chat'

const increment = () => {
  counter.value++
}

// --- REPLICATION & CONNEXION ---
const startLiveReplication = (db: PouchDB.Database) => {
  if (replicationHandle) return
  replicationHandle = db.sync(COUCH_DB_URL, { live: true, retry: true })
    .on('change', () => fetchData()) // Rafraîchir à chaque changement
    .on('error', (err) => console.error('Erreur sync:', err))
}

const initDatabase = () => {
  console.log('=> Connexion DB')
  const localDB = new PouchDB('Posts')
  storage.value = localDB
  if (!isOffline.value) {
    startLiveReplication(localDB)
  }
}

const toggleOfflineMode = () => {
  isOffline.value = !isOffline.value
  if (isOffline.value) {
    if (replicationHandle) {
      replicationHandle.cancel()
      replicationHandle = null
    }
    console.log("Offline")
  } else {
    if (storage.value) {
      startLiveReplication(storage.value)
      MAJserveur()
    }
    console.log("Online")
  }
}

const MAJserveur = async () => {
  const remoteDB = new PouchDB(COUCH_DB_URL)
  try {
    await storage.value?.replicate.to(remoteDB)
    await storage.value?.replicate.from(remoteDB)
    fetchData()
  } catch (err) {
    console.error('Erreur sync manuelle:', err)
  }
}

// --- CRUD POSTS DE BASE ---
const fetchData = () => {
  // Par défaut, on récupère tout. 
  // Note: Pour le tri, on utilisera sortByLikes explicitement.
  storage.value?.allDocs({ include_docs: true, attachments: true })
    .then((result: any) => {
      postsData.value = result.rows.map((row: any) => row.doc)
    })
    .catch((err: any) => console.log(err))
}

const createDoc = (newDoc: any) => {
  // Initialiser les nouveaux champs
  newDoc.likes = 0
  newDoc.comments = []
  
  storage.value?.post(newDoc)
    .then(() => fetchData())
    .catch((err: any) => console.log(err))
}

const updateDoc = (doc: any) => {
  const newDoc = { ...doc }
  newDoc.ville = 'Update ' + new Date().toISOString() // Exemple de modif
  storage.value?.put(newDoc)
    .then(() => fetchData())
    .catch((err: any) => console.log(err))
}

const deleteDoc = (doc: any) => {
  storage.value?.remove(doc)
    .then(() => fetchData())
    .catch((err: any) => console.log(err))
}

// --- FEATURES AJOUTÉES ---

// 1. LIKES
const toggleLike = (post: any) => {
  const updated = { ...post, likes: (post.likes || 0) + 1 }
  storage.value?.put(updated)
    .then(() => fetchData()) // On pourrait optimiser, mais fetchData suffit
    .catch((e) => console.error(e))
}

// 2. TRI PAR LIKES (DB SIDE)
const sortByLikes = () => {
  // Utilise l'index pour trier sans TS
  storage.value?.find({
    selector: { likes: { $exists: true } },
    sort: [{ likes: 'desc' }]
  }).then((res: any) => {
    postsData.value = res.docs
    console.log("Trié par likes via DB")
  }).catch((e) => console.error(e))
}

// 3. COMMENTAIRES (Ajout / Modif / Suppr)
const addComment = (post: any) => {
  const txt = prompt("Commentaire :")
  if (!txt) return
  
  const newCom: Comment = {
    id: Date.now().toString(),
    text: txt,
    date: new Date().toISOString()
  }
  
  const updated = { ...post, comments: [...(post.comments || []), newCom] }
  storage.value?.put(updated).then(() => fetchData())
}

const editComment = (post: any, comment: Comment) => {
  const newTxt = prompt("Modifier commentaire :", comment.text)
  if (!newTxt) return

  // On met à jour le tableau localement puis on envoie
  const updatedComments = post.comments.map((c: Comment) => {
    if (c.id === comment.id) return { ...c, text: newTxt }
    return c
  })
  
  const updated = { ...post, comments: updatedComments }
  storage.value?.put(updated).then(() => fetchData())
}

const deleteComment = (post: any, commentId: string) => {
  const updatedComments = post.comments.filter((c: Comment) => c.id !== commentId)
  const updated = { ...post, comments: updatedComments }
  storage.value?.put(updated).then(() => fetchData())
}

// --- FACTORY & INDEX & RECHERCHE ---

const createFactoryDocs = () => {
  if (!storage.value) return
  const docsToAdd: Post[] = []
  const sports = ['Football', 'Tennis', 'Rugby']
  
  for (let i = 0; i < 20; i++) {
    const doc: Post = {
      nom: `User ${i + 1}`,
      age: 20 + i,
      ville: `Ville ${i}`,
      sport: sports[i % 3],
      likes: Math.floor(Math.random() * 50), // Random likes
      comments: []
    }
    docsToAdd.push(doc)
  }
  storage.value.bulkDocs(docsToAdd).then(() => fetchData())
}

const createIndex = () => {
  // Index pour le tri par likes
  storage.value?.createIndex({ index: { fields: ['likes'] } })
  // Index pour la recherche par nom
  storage.value?.createIndex({ index: { fields: ['nom'] } })
}

const searchByName = () => {
  if (!searchTerm.value) {
    fetchData()
    return
  }
  // Recherche DB via Regex
  const regex = new RegExp(`^${searchTerm.value}`, 'i')
  storage.value?.find({
    selector: { nom: { $regex: regex } }
  }).then((res: any) => {
    postsData.value = res.docs
  })
}

onMounted(() => {
  initDatabase()
  createIndex()
  fetchData()
})
</script>

<template>
  <h1>TP Infra Données</h1>
  <p>Compteur: {{ counter }} <button @click="increment">+1</button></p>
  
  <hr>
  
  <div>
    <h2>Connexion</h2>
    <p>Etat: {{ isOffline ? 'HORS LIGNE' : 'EN LIGNE' }}</p>
    <button @click="toggleOfflineMode">Basculer Mode</button>
    <button @click="MAJserveur">Sync Manuelle</button>
  </div>

  <hr>

  <div>
    <h2>Outils</h2>
    <button @click="createFactoryDocs">Factory (20 docs)</button>
    <br>
    <button @click="sortByLikes">Trier par Likes (via DB)</button>
    <br>
    <input v-model="searchTerm" placeholder="Nom..." >
    <button @click="searchByName">Rechercher</button>
    <button @click="fetchData">Reset</button>
  </div>

  <hr>

  <div>
    <h2>Liste des Posts</h2>
    <ul>
      <li v-for="post in postsData" :key="post._id">
        
        <h3>{{ post.nom }} ({{ post.likes }} likes)</h3>
        <p>{{ post.ville }} - {{ post.sport }}</p>
        
        <button @click="toggleLike(post)">Liker</button>
        <button @click="updateDoc(post)">Modifier Post</button>
        <button @click="deleteDoc(post)">Supprimer Post</button>
        <button @click="addComment(post)">Ajouter Commentaire</button>

        <div v-if="post.comments && post.comments.length > 0">
           <h4>Commentaires:</h4>
           <ul>
             <li v-for="comment in post.comments" :key="comment.id">
               {{ comment.text }}
               <button @click="editComment(post, comment)">Edit</button>
               <button @click="deleteComment(post, comment.id)">Suppr</button>
             </li>
           </ul>
        </div>
        <hr>
      </li>
    </ul>
  </div>

  <button @click="createDoc({ nom: 'Nuno', age: 22, ville: 'Lausanne', sport: 'foot' })">
    Créer Document Test
  </button>
</template>