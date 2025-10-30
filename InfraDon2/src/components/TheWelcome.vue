<!-- <script setup lang="ts">
import { ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'

// Déclaration du type Post
interface Post {
  _id?: string // facultatif, sera généré automatiquement
  post_name: string
  post_content: string
  attributes: {
    creation_date: any
  }
}

// Référence à la base de données
const storage = ref<any>()

// Données récupérées depuis la base
const postsData = ref<Post[]>([])

// Champs pour le formulaire
const postName = ref('')
const postContent = ref('')

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données')

  const db = new PouchDB('http://nuno:Nunofaria17@localhost:5984/infradonn2_chat')

  if (db) {
    console.log('✅ Connecté à la collection :', db.name)
    storage.value = db

    // Une fois la base prête, on récupère les données
    fetchData()
  } else {
    console.warn('⚠️ Échec de la connexion à la base de données')
  }
}

// --- Version prof intégrée ici ---
const fetchData = (): any => {
  storage.value
    .allDocs({ include_docs: true })
    .then((result: any) => {
      console.log('=> Données récupérées :', result.rows)
      postsData.value = result.rows.map((row: any) => row.doc)
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des données :', error)
    })
}
// -------------------------------------

const addMultiplePosts = async (newPosts: Post[]) => {
  if (!storage.value) {
    console.warn('Base de données non initialisée.')
    return
  }

  try {
    // Chaque doc doit avoir un _id unique
    const docsToAdd = newPosts.map((post) => ({
      _id: 'post_' + new Date().toISOString() + '_' + Math.random().toString(36).substr(2, 9),
      ...post,
    }))

    const response = await storage.value.bulkDocs(docsToAdd)
    console.log('Documents ajoutés :', response)

    fetchData() // rafraîchir la liste après ajout
  } catch (error) {
    console.error("Erreur lors de l'ajout multiple :", error)
  }
}

// Gestion du formulaire pour ajouter un post
const handleSubmit = () => {
  if (!postName.value.trim() || !postContent.value.trim()) {
    alert('Merci de remplir tous les champs !')
    return
  }

  const newPost: Post = {
    post_name: postName.value,
    post_content: postContent.value,
    attributes: {
      creation_date: new Date().toISOString(),
    },
  }

  addMultiplePosts([newPost])

  // Réinitialiser les champs du formulaire
  postName.value = ''
  postContent.value = ''
}

// Appelé à l'initialisation du composant
onMounted(() => {
  console.log('🔄 Composant initialisé')
  initDatabase()
})
</script>

<template>
  <h1>🗃️ Liste des posts</h1>

  <form @submit.prevent="handleSubmit" style="margin-bottom: 20px">
    <input
      v-model="postName"
      placeholder="Nom du post"
      required
      style="display: block; margin-bottom: 10px; width: 300px; padding: 5px"
    />
    <textarea
      v-model="postContent"
      placeholder="Contenu du post"
      required
      rows="4"
      style="display: block; margin-bottom: 10px; width: 300px; padding: 5px"
    ></textarea>
    <button type="submit">Ajouter un post</button>
  </form>

  <article v-for="post in postsData" :key="post._id">
    <h2>{{ post.post_name }}</h2>
    <p>{{ post.post_content }}</p>
  </article>
</template> -->
*/

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
  _id?: string
  _rev?: string
}

// Référence à la base de données
const storage = ref()
// Données stockées
const postsData = ref<Post[]>([])

// Initialisation de la base de données
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

// Récupération des données
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

  // TODO Récupération des données
  // https://pouchdb.com/api.html#batch_fetch
  // Regarder l'exemple avec function allDocs
  // Remplir le tableau postsData avec les données récupérées
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
