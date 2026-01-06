<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'
PouchDB.plugin(PouchDBFind)


declare interface CommentDoc {
  _id?: string
  _rev?: string
  postId: string
  text: string
  date: string
}

declare interface Post {
  _id?: string
  _rev?: string
  nom: string
  age: number
  ville: string
  sport: string
  likes: number
  displayComments?: CommentDoc[]

  //gerer les medias
  _attachments?: any
  imageSrc?: string
  
  //pour afficher ou pas tous les comms
  showAllComments?: boolean 
}


const counter = ref(0)
const postsDB = ref<PouchDB.Database | undefined>()
const commentsDB = ref<PouchDB.Database | undefined>()

const postsData = ref<Post[]>([])
const isOffline = ref(false)
const searchTerm = ref('')

const paginationOffset = ref(0) 

let postSyncHandler: PouchDB.Replication.Sync<any> | null = null
let commentSyncHandler: PouchDB.Replication.Sync<any> | null = null

const COUCH_URL_POSTS = 'http://nuno:Nunofaria17@localhost:5984/infradonn2_chat'
const COUCH_URL_COMMENTS = 'http://nuno:Nunofaria17@localhost:5984/infradonn2_comments'

const increment = () => counter.value++


const initDatabases = () => {
  console.log('=> Init des 2 Collections')
  postsDB.value = new PouchDB('Posts')
  commentsDB.value = new PouchDB('Comments')

  if (!isOffline.value) {
    startLiveReplication()
  }
}

const startLiveReplication = () => {
  if (postSyncHandler || !postsDB.value || !commentsDB.value) return
  postSyncHandler = postsDB.value.sync(COUCH_URL_POSTS, { live: true, retry: true }).on('change', () => fetchTop10(false))
  commentSyncHandler = commentsDB.value.sync(COUCH_URL_COMMENTS, { live: true, retry: true }).on('change', () => fetchTop10(false))
}

const toggleOfflineMode = () => {
  isOffline.value = !isOffline.value
  if (isOffline.value) {
    postSyncHandler?.cancel(); postSyncHandler = null;
    commentSyncHandler?.cancel(); commentSyncHandler = null;
    console.log('Mode Offline')
  } else {
    startLiveReplication()
    console.log('Mode Online')
  }
}

const MAJserveur = async () => {
  try {
    await postsDB.value?.replicate.to(COUCH_URL_POSTS)
    await postsDB.value?.replicate.from(COUCH_URL_POSTS)
    await commentsDB.value?.replicate.to(COUCH_URL_COMMENTS).catch(e => console.log('Pas de DB remote comments'))
    await commentsDB.value?.replicate.from(COUCH_URL_COMMENTS).catch(e => console.log('Pas de DB remote comments'))
    fetchTop10(false) // Refresh
  } catch (err) { console.error('Erreur sync:', err) }
}

const processPostsForDisplay = async (rawPosts: Post[]) => {
  if (!commentsDB.value || !postsDB.value) return []

  //On charge les comms
  const allCommentsRes = await commentsDB.value.allDocs({ include_docs: true })
  const comments = allCommentsRes.rows.map((row: any) => row.doc)

  return await Promise.all(rawPosts.map(async (post: Post) => {
      // Jointure
      const linkedComments = comments.filter((c: CommentDoc) => c.postId === post._id)
      
      let urlImage = ''
      if (post._attachments && post._attachments['image']) {
        try {
          const blob = await postsDB.value!.getAttachment(post._id!, 'image') as Blob
          urlImage = URL.createObjectURL(blob)
        } catch (e) { console.log('Erreur image', e) }
      }

      return { 
          ...post, 
          displayComments: linkedComments, 
          imageSrc: urlImage,
          showAllComments: false 
      }
  }))
}


/**
 * MOTIVATION de mon choix par rapport a allDocs :
 * Au lieu d'utiliser allDocs() qui charge en peu toute la base en mémoire,
 * je veux utiliser find()
 */
const fetchTop10 = async (append = false) => {
  if (!postsDB.value) return

  
  if (!append) paginationOffset.value = 0

  try {
    const res = await postsDB.value.find({
        selector: { likes: { $gte: 0 } }, 
        sort: [{ likes: 'desc' }], 
        limit: 10,                
        skip: paginationOffset.value 
    })

    const processedPosts = await processPostsForDisplay(res.docs as Post[])

    if (append) {
        postsData.value = [...postsData.value, ...processedPosts]
    } else {
        postsData.value = processedPosts
    }
    
  } catch (err) { 
      console.error('Erreur fetchTop10 (Index manquant ?):', err) 
  }
}

const loadNextPage = () => {
    paginationOffset.value += 10 
    fetchTop10(true) 
}



const createDoc = (newDoc: any) => {
  newDoc.likes = 0
  postsDB.value?.post(newDoc).then(() => fetchTop10(false))
}
const deleteDoc = (doc: Post) => {
  postsDB.value?.remove(doc._id!, doc._rev!).then(() => fetchTop10(false))
}
const updateDoc = (doc: Post) => {
  const { displayComments, imageSrc, showAllComments, ...docToSave } = doc
  const docUpdated = { ...docToSave, ville: 'Update ' + new Date().toISOString() }
  postsDB.value?.put(docUpdated).then(() => fetchTop10(false))
}
const toggleLike = (post: Post) => {
  const { displayComments, imageSrc, showAllComments, ...docToSave } = post
  const updated = { ...docToSave, likes: (docToSave.likes || 0) + 1 }
  postsDB.value?.put(updated).then(() => fetchTop10(false))
}

//add un comment a un post
const addComment = (post: Post) => {
  const txt = prompt('Commentaire :')
  if (!txt || !commentsDB.value) return
  const newComment: CommentDoc = { postId: post._id!, text: txt, date: new Date().toISOString() }
  commentsDB.value.post(newComment).then(() => fetchTop10(false))
}

const editComment = (comment: CommentDoc) => {
  if (!commentsDB.value || !comment._id || !comment._rev) return
  const newText = prompt('Modifier le commentaire :', comment.text)
  if (newText === null || newText.trim() === '') return
  const updatedComment = { ...comment, text: newText }
  commentsDB.value.put(updatedComment).then(() => fetchTop10(false))
}

const deleteComment = (comment: CommentDoc) => {
  if (!commentsDB.value || !comment._id || !comment._rev) return
  commentsDB.value.remove(comment._id, comment._rev).then(() => fetchTop10(false))
}

//partie des medias
const attachImage = async (event: any, post: Post) => {
  const file = event.target.files[0]
  if (!file || !postsDB.value || !post._id || !post._rev) return
  try {
    await postsDB.value.putAttachment(post._id, 'image', post._rev, file, file.type)
    fetchTop10(false)
  } catch (err) { console.error('Erreur upload:', err) }
}

const deleteImage = async (post: Post) => {
  if (!postsDB.value || !post._id || !post._rev) return
  try {
    await postsDB.value.removeAttachment(post._id, 'image', post._rev)
    fetchTop10(false)
  } catch (err) { console.error('Erreur suppr image:', err) }
}

//indexation pour faire la recherche
const createIndex = () => {
  if (!postsDB.value) return
  postsDB.value.createIndex({ index: { fields: ['likes'] } })
    .then(() => postsDB.value?.createIndex({ index: { fields: ['nom'] } }))
    .then(() => console.log('✅ Index créés (Tri et Recherche optimisés)'))
    .catch(err => console.error('Erreur Index', err))
}

const searchByName = () => {
  if (!searchTerm.value) { fetchTop10(false); return }
  
  const regex = new RegExp(`^${searchTerm.value}`, 'i')
  postsDB.value?.find({ selector: { nom: { $regex: regex } } })
    .then(async (res: any) => { 
       postsData.value = await processPostsForDisplay(res.docs)
    })
}

const createFactoryDocs = async () => {
  if (!postsDB.value || !commentsDB.value) return
  const postsToAdd: any[] = []
  const commentsToAdd: any[] = []
  const sports = ['Football', 'Tennis', 'Rugby']

  
  for (let i = 0; i < 25; i++) {
    const postId = 'post_fact_' + Date.now() + '_' + i
    postsToAdd.push({
      _id: postId,
      nom: `User ${i + 1}`,
      age: 20 + i,
      ville: `Ville ${i}`,
      sport: sports[i % 3],
      likes: Math.floor(Math.random() * 200), 
    })

    commentsToAdd.push({ postId: postId, text: `Com 1 (Vieux) - Post ${i}`, date: '2023-01-01' })
    commentsToAdd.push({ postId: postId, text: `Com 2 (Moyen) - Post ${i}`, date: '2023-01-02' })
    commentsToAdd.push({ postId: postId, text: `Com 3 (Dernier) - Post ${i}`, date: '2023-01-03' })
  }
  await postsDB.value.bulkDocs(postsToAdd)
  await commentsDB.value.bulkDocs(commentsToAdd)
  console.log('Factory OK')
  fetchTop10(false)
}

onMounted(() => {
  initDatabases()
  createIndex()
  fetchTop10(false) //du coup recharger que le top10 au demarage
})
</script>

<template>
  <h1>Tp InfraDon - WhatsApp</h1>
  <p>Compteur: {{ counter }} <button @click="increment">+1</button></p>

  <hr />

  <div>
    <h2>Connexion</h2>
    <p>Etat: <strong>{{ isOffline ? 'HORS LIGNE' : 'EN LIGNE' }}</strong></p>
    <button @click="toggleOfflineMode">Online / Offline</button>
    <button @click="MAJserveur">Sync Manuelle</button>
  </div>

  <hr />

  <div>
    <h2>Outils</h2>
    <button @click="createFactoryDocs">Factory (25 Docs)</button>
    <br />
    <input v-model="searchTerm" placeholder="Nom..." >
    <button @click="searchByName">Rechercher</button>
    <button @click="fetchTop10(false)">Reset / Top 10</button>
  </div>

  <hr />

  <div>
    <h2>Top Posts (Triés par Likes)</h2>
    <ul>
      <li v-for="post in postsData" :key="post._id">
        <h3>{{ post.nom }} ({{ post.likes }} likes)</h3>
        <p>{{ post.ville }} - {{ post.sport }}</p>

        <div style="margin: 10px 0">
          <div v-if="post.imageSrc">
            <img :src="post.imageSrc" style="max-width: 200px; border-radius: 8px" />
            <br />
            <button @click="deleteImage(post)">Supprimer Image</button>
          </div>
          <div v-else>
            <label>Photo : </label>
            <input type="file" @change="(event) => attachImage(event, post)" accept="image/*" />
          </div>
        </div>

        <button @click="toggleLike(post)">Like</button>
        <button @click="updateDoc(post)">Edit</button>
        <button @click="deleteDoc(post)">Suppr</button>
        <button @click="addComment(post)">Commenter</button>

        <div v-if="post.displayComments && post.displayComments.length > 0" style="background: #f4f4f4; padding: 10px; margin-top: 10px;">
          
          <div style="display: flex; justify-content: space-between; align-items: center;">
             <h4>Commentaires :</h4>
             <button v-if="post.displayComments.length > 1" @click="post.showAllComments = !post.showAllComments">
                {{ post.showAllComments ? 'Masquer' : `Voir les ${post.displayComments.length} commentaires` }}
             </button>
          </div>

          <ul>
            <li v-for="comment in (post.showAllComments ? post.displayComments : post.displayComments.slice(-1))" :key="comment._id">
              
              <strong>{{ comment.text }}</strong> <small>({{ comment.date }})</small>
              
              <span v-if="!post.showAllComments && post.displayComments.length > 1" style="color: grey; font-size: 0.8em;"> (Dernier commentaire)</span>

              <div style="margin-top: 5px;">
                 <button @click="editComment(comment)">Modifier</button>
                 <button @click="deleteComment(comment)">Suppr</button>
              </div>
            </li>
          </ul>
        </div>
        <hr />
      </li>
    </ul>

    <div v-if="postsData.length > 0" style="text-align: center; margin: 20px;">
        <button @click="loadNextPage" style="padding: 10px 20px; font-size: 1.1em; cursor: pointer;">
             Charger les 10 suivants 
        </button>
    </div>

  </div>

  <button @click="createDoc({ nom: 'Nuno', age: 22, ville: 'Lausanne', sport: 'foot' })">
    Créer Document
  </button>
</template>