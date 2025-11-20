<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'
PouchDB.plugin(PouchDBFind)

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
  likes: number
  comments: Comment[]
  _id?: string
  _rev?: string
}

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

const startLiveReplication = (db: PouchDB.Database) => {
  if (replicationHandle) return
  replicationHandle = db
    .sync(COUCH_DB_URL, { live: true, retry: true })
    .on('change', () => fetchData())
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
    console.log('Offline')
  } else {
    if (storage.value) {
      startLiveReplication(storage.value)
      MAJserveur()
    }
    console.log('Online')
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
const fetchData = () => {
  storage.value
    ?.allDocs({ include_docs: true, attachments: true })
    .then((result: any) => {
      postsData.value = result.rows.map((row: any) => row.doc)
    })
    .catch((err: any) => console.log(err))
}

const createDoc = (newDoc: any) => {
  newDoc.likes = 0
  newDoc.comments = []

  storage.value
    ?.post(newDoc)
    .then(() => fetchData())
    .catch((err: any) => console.log(err))
}

const updateDoc = (doc: any) => {
  const newDoc = { ...doc }
  newDoc.ville = 'Update ' + new Date().toISOString()
  storage.value
    ?.put(newDoc)
    .then(() => fetchData())
    .catch((err: any) => console.log(err))
}

const deleteDoc = (doc: any) => {
  storage.value
    ?.remove(doc)
    .then(() => fetchData())
    .catch((err: any) => console.log(err))
}

const toggleLike = (post: any) => {
  const updated = { ...post, likes: (post.likes || 0) + 1 }
  storage.value
    ?.put(updated)
    .then(() => fetchData())
    .catch((e) => console.error(e))
}

const sortByLikes = () => {
  if (!storage.value) return


  storage.value
    .find({
      selector: {
        likes: { $gte: 0 },
      },
      sort: [{ likes: 'desc' }],
    })
    .then((result: any) => {
      console.log(`Tri réussi : ${result.docs.length} documents.`)
      postsData.value = result.docs
    })
    .catch((err: any) => {
      console.error('Erreur Tri:', err)
    })
}

const addComment = (post: any) => {
  const txt = prompt('Commentaire :')
  if (!txt) return

  const newCom: Comment = {
    id: Date.now().toString(),
    text: txt,
    date: new Date().toISOString(),
  }

  const updated = { ...post, comments: [...(post.comments || []), newCom] }
  storage.value?.put(updated).then(() => fetchData())
}

const editComment = (post: any, comment: Comment) => {
  const newTxt = prompt('Modifier commentaire :', comment.text)
  if (!newTxt) return

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
      likes: 0, // chaque nouveau post du coup commence avec 0 likes
      comments: [],
    }
    docsToAdd.push(doc)
  }
  storage.value.bulkDocs(docsToAdd).then(() => fetchData())
}

const createIndex = () => {
  storage.value
    ?.createIndex({
      index: { fields: ['nom'] },
    })
    .then(() => {
      return storage.value?.createIndex({
        index: { fields: ['likes'] },
      })
    })
    .catch((err) => {
      console.error('Erreur Index:', err)
    })
}

const searchByName = () => {
  if (!searchTerm.value) {
    fetchData()
    return
  }
  const regex = new RegExp(`^${searchTerm.value}`, 'i')
  storage.value
    ?.find({
      selector: { nom: { $regex: regex } },
    })
    .then((res: any) => {
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

  <hr />

  <div>
    <h2>Connexion</h2>
    <p>Etat: {{ isOffline ? 'HORS LIGNE' : 'EN LIGNE' }}</p>
    <!--Juste pour savoir si je suis en mode online ou pas-->
    <button @click="toggleOfflineMode">Changer mode Online : Offline</button>
    <button @click="MAJserveur">MAJ manuellement</button>
  </div>

  <hr />

  <div>
    <h2>Outils</h2>
    <button @click="createFactoryDocs">Factory (faire 20 docs randoms)</button>
    <br />
    <button @click="sortByLikes">Trier par Likes</button>
    <br />
    <input v-model="searchTerm" placeholder="Nom..." />
    <button @click="searchByName">Rechercher</button>
    <button @click="fetchData">Trie de base</button>
  </div>

  <hr />

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
              <button @click="editComment(post, comment)">Modifier</button>
              <button @click="deleteComment(post, comment.id)">X</button>
            </li>
          </ul>
        </div>
        
      </li>
    </ul>
  </div>
  <hr></hr>
  <button @click="createDoc({ nom: 'Nuno', age: 22, ville: 'Lausanne', sport: 'foot' })">
    Créer Document Test
  </button>
</template>
