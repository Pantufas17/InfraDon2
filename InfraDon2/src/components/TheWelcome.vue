<script setup lang="ts">
import { onMounted, ref } from 'vue'
// Importez PouchDB Find pour la recherche indexée (important pour le TP)
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'
PouchDB.plugin(PouchDBFind) // Active le plugin find

// --- Interfaces et Types ---
declare interface Post {
    nom: string
    age: number
    ville: string
    sport: string
    _id?: string
    _rev?: string
}

// --- Références ---
const counter = ref(0)
const storage = ref<PouchDB.Database | undefined>()
const postsData = ref<Post[]>([])
const isOffline = ref(false)
const searchTerm = ref('') // NOUVEAU : Pour la recherche indexée

// NOUVEAU : Référence pour gérer l'arrêt de la réplication continue
let replicationHandle: PouchDB.Replication.Sync<Post> | null = null
const COUCH_DB_URL = 'http://nuno:Nunofaria17@localhost:5984/infradonn2_chat' 


const increment = () => {
    counter.value++
}

// NOUVEAU : Fonction pour démarrer la réplication live bidirectionnelle (sync)
const startLiveReplication = (db: PouchDB.Database) => {
    if (replicationHandle) return // Évite de démarrer deux fois

    replicationHandle = db.sync(COUCH_DB_URL, {
        live: true, 
        retry: true, 
    })
    .on('change', () => {
        // Rafraîchir la liste à chaque changement répliqué
        fetchData() 
    })
    .on('error', (err) => {
        console.error('Erreur de synchronisation live:', err)
    })
}


const initDatabase = () => {
    console.log('=> Connexion à la base de données')
    
    const localDB = new PouchDB('Posts')
    storage.value = localDB
    console.log('Base locale :', localDB?.name)

    // Remplace l'ancien bloc replicate.from par le lancement du sync
    if (!isOffline.value) {
        startLiveReplication(localDB)
    }
}

const fetchData = () => {
    storage.value?.allDocs({
        include_docs: true,
        attachments: true,
    })
    .then((result: any) => {
        postsData.value = result.rows.map((row: any) => row.doc)
        console.log('Documents récupérés:', postsData.value.length)
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
            // Optionnel : mise à jour locale pour éviter le fetchData lent
            // doc._rev = response.rev 
            // doc.ville = newDoc.ville
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

//online / ofline mode (mis à jour pour annuler/redémarrer la synchronisation)
const toggleOfflineMode = () => {
    isOffline.value = !isOffline.value
    if (isOffline.value) {
        console.log("Mode offline activé. Arrêt de la synchronisation.")
        if (replicationHandle) {
            replicationHandle.cancel() // Arrête la synchro live
            replicationHandle = null
        }
    } else {
        console.log("Mode online activé. Redémarrage de la synchronisation.")
        if (storage.value) {
            startLiveReplication(storage.value) // Redémarre la synchro live
        }
    }
}

const MAJserveur = async () => {
    const remoteDB = new PouchDB(COUCH_DB_URL);
    
    try {
      
        await storage.value?.replicate.to(remoteDB); 
        console.log('Données envoyées au serveur');

        await storage.value?.replicate.from(remoteDB); 
        console.log('Données récupérées du serveur');
        
        fetchData();
    } catch (err) {
        console.error('Erreur de synchronisation :(', err);
    }
};

// nouvelle partie factory pour generer du cuop et alimenter d un coup ma db avec des truxs random
const createFactoryDocs = () => {
    if (!storage.value) return
    
    const docsToAdd: Post[] = []
    const sports = ['Football', 'Basketball', 'Tennis', 'Natation', 'Course', 'Escrime', 'Rugby', 'Volley']
    
    for (let i = 0; i < 50; i++) {
        const doc: Post = {
            nom: `Factory User ${i + 1}`,
            age: Math.floor(Math.random() * 50) + 18,
            ville: `Ville ${Math.floor(Math.random() * 10)}`,
            sport: sports[Math.floor(Math.random() * sports.length)],
        }
        docsToAdd.push(doc)
    }

    storage.value.bulkDocs(docsToAdd)
        .then(() => {
            console.log(`✅ ${docsToAdd.length} documents ajoutés par la factory.`)
            fetchData()
        })
        .catch(err => {
            console.error('Erreur bulkDocs:', err)
        })
}


//13.11 creation index => Aide utilisation IA car je n'arrivais pas, ca me faisait des erreurs a chaque fois
const createIndex = () => {
    storage.value?.createIndex({
        index: { fields: ['nom'] } 
    }).then((result) => {
        console.log('Index créé sur le champ "nom"', result);
    }).catch((err) => {
        console.error('Erreur lors de la création de l’index', err);
    })
}

//Egalement aide IA car, quand je faisait la recherche ca ne marchait pas. Ca faisait pas match
const searchByName = () => {
    if (!storage.value || !searchTerm.value.trim()) {
        fetchData()
        return
    }
    
 
    const regex = new RegExp(`^${searchTerm.value.trim()}`, 'i'); 

    storage.value?.find({
        selector: { 
            nom: { $regex: regex } 
        },
        fields: ['_id', 'nom', 'age', 'ville', 'sport']
    }).then((result: any) => {
        console.log('Résultats de la recherche floue:', result.docs.length)
        postsData.value = result.docs
    }).catch((err) => {
        console.error('Erreur lors de la recherche', err)
    })
}

onMounted(() => {
    console.log('=> Composant initialisé')
    initDatabase()  
    createIndex()  
})
</script>

<template>
  <h1>Posts - Pratique InfraDon2</h1>
  <p>Counter: {{ counter }}</p>
  <button @click="increment">+1</button>
  
  <hr>

      <div>
    <h2>1. Gestion Offline / Réplication</h2>
    <p>Statut : <strong :style="{ color: isOffline ? 'darkorange' : 'green' }">
        {{ isOffline ? 'HORS LIGNE' : 'EN LIGNE' }}
    </strong></p>
    
    <button @click="toggleOfflineMode" style="margin-right: 10px;">
      {{ isOffline ? 'Passer en mode Online' : 'Passer en mode Offline' }}
    </button>
    
    <button @click="MAJserveur" :disabled="!isOffline">
        Synchronisation Manuelle (Envoyer/Récupérer)
    </button>
  </div>
    <hr>
    
    <div>
        <h2>2. Factory et Recherche Indexée</h2>
        <button @click="createFactoryDocs" style="margin-bottom: 10px;">
            Factory : Générer 50 documents
        </button>
        <br>
        <input 
            type="text" 
            v-model="searchTerm" 
            placeholder="Rechercher par nom (indexé)" 
            style="margin-right: 5px;"
        />
        <button @click="searchByName">
            Rechercher
        </button>
        <button @click="fetchData" style="margin-left: 10px;">
            Afficher tout
        </button>
    </div>
    <hr>
  
      <p>PostDatas</p>
  <ul>
    <li v-for="post in postsData" :key="post._id" style="border-bottom: 1px dotted #ccc; padding: 5px 0;">
      {{ post.nom }} - {{ post.age }} - {{ post.ville }} [{{ post.sport }}]
      <button @click="updateDoc(post)">Modifier</button>
      <button @click="deleteDoc(post)">Supprimer</button>
    </li>
  </ul>
  
  <button @click="createDoc({ nom: 'Nuno', age: 22, ville: 'Lausanne', sport: 'foot' })">
    Créer un document
  </button>
</template>