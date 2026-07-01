<template>
  <div class="carnet-app">
    <header>
      <h1> Carnet d'Adresses</h1>
      <p>{{ contactsActifs.length }} contact(s) au total</p>
    </header>

    <!-- Onglets -->
    <div class="tabs">
      <button :class="{ active: onglet === 'contacts' }" @click="onglet = 'contacts'">
        Contacts
      </button>
      <button :class="{ active: onglet === 'corbeille' }" @click="onglet = 'corbeille'">
         Corbeille ({{ contactsCorbeille.length }})
      </button>
    </div>

    <!-- ===================== VUE CONTACTS ===================== -->
    <div v-if="onglet === 'contacts'">
      <!-- Formulaire d'ajout / modification -->
      <form class="contact-form" @submit.prevent="validerFormulaire">
        <div class="form-row">
          <input type="text" v-model="formulaire.prenom" placeholder="Prénom">
          <input type="text" v-model="formulaire.nom" placeholder="Nom">
        </div>
        <div class="form-row">
          <input type="tel" v-model="formulaire.telephone" placeholder="Téléphone">
          <input type="email" v-model="formulaire.email" placeholder="Email (optionnel)">
        </div>
        <div class="form-actions">
          <button type="submit" :disabled="!peutValider">
            {{ idEnEdition ? 'Enregistrer' : 'Ajouter le contact' }}
          </button>
          <button type="button" class="btn-annuler" v-if="idEnEdition" @click="annulerEdition">
            Annuler
          </button>
        </div>
      </form>

      <!-- Liste triée alphabétiquement -->
      <ul class="contact-list" v-if="contactsTries.length">
        <li v-for="contact in contactsTries" :key="contact.id" class="contact-item">
          <div class="contact-avatar">{{ initiales(contact) }}</div>
          <div class="contact-infos">
            <span class="contact-nom">{{ contact.nom }} {{ contact.prenom }}</span>
            <span class="contact-detail">{{ contact.telephone }}</span>
            <span class="contact-detail" v-if="contact.email">{{ contact.email }}</span>
          </div>
          <div class="contact-actions">
            <button class="icon-btn" @click="modifierContact(contact)" title="Modifier">✎</button>
            <button class="icon-btn delete" @click="mettreALaCorbeille(contact.id)" title="Supprimer">✕</button>
          </div>
        </li>
      </ul>

      <div class="empty-state" v-else>
        Aucun contact pour l'instant...remplisser les champs ci dessus,merci.
      </div>
    </div>

    <!-- ===================== VUE CORBEILLE ===================== -->
    <div v-else>
      <ul class="contact-list" v-if="contactsCorbeille.length">
        <li v-for="contact in contactsCorbeille" :key="contact.id" class="contact-item corbeille-item">
          <div class="contact-avatar muted">{{ initiales(contact) }}</div>
          <div class="contact-infos">
            <span class="contact-nom">{{ contact.nom }} {{ contact.prenom }}</span>
            <span class="contact-detail">{{ contact.telephone }}</span>
            <span class="contact-detail expiration">
              Suppression définitive dans {{ joursRestants(contact) }} jour(s)
            </span>
          </div>
          <div class="contact-actions">
            <button class="icon-btn restore" @click="restaurerContact(contact.id)" title="Restaurer">↺</button>
          </div>
        </li>
      </ul>

      <div class="empty-state" v-else>
        La corbeille est vide 🗑️
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue'

const CLE_STOCKAGE = 'carnet-adresses-contacts'
const DUREE_CORBEILLE_MS = 5 * 24 * 60 * 60 * 1000 // 5 jours en millisecondes

// --- État réactif ---
const contacts = ref([])
const onglet = ref('contacts') // 'contacts' | 'corbeille'
const idEnEdition = ref(null)  // id du contact en cours de modification, ou null

const formulaire = ref({
  prenom: '',
  nom: '',
  telephone: '',
  email: ''
})

// --- Charger les contacts au démarrage ---
onMounted(() => {
  const sauvegarde = localStorage.getItem(CLE_STOCKAGE)
  if (sauvegarde) {
    contacts.value = JSON.parse(sauvegarde)
  }
  purgerCorbeilleExpiree()
})

// --- Sauvegarder automatiquement à chaque changement ---
watch(contacts, (nouveauxContacts) => {
  localStorage.setItem(CLE_STOCKAGE, JSON.stringify(nouveauxContacts))
}, { deep: true })

// --- Supprime définitivement les contacts dont les 5 jours sont écoulés ---
function purgerCorbeilleExpiree() {
  const maintenant = Date.now()
  contacts.value = contacts.value.filter(c => {
    if (!c.supprimeLe) return true // contact actif, on garde
    return (maintenant - c.supprimeLe) < DUREE_CORBEILLE_MS
  })
}

// --- Validation simple du formulaire ---
const peutValider = computed(() => {
  return formulaire.value.prenom.trim() && formulaire.value.nom.trim() && formulaire.value.telephone.trim()
})

// --- Ajouter ou enregistrer une modification ---
function validerFormulaire() {
  if (!peutValider.value) return

  if (idEnEdition.value) {
    // Mode modification : on retrouve le contact et on met à jour ses champs
    const contact = contacts.value.find(c => c.id === idEnEdition.value)
    contact.prenom = formulaire.value.prenom.trim()
    contact.nom = formulaire.value.nom.trim()
    contact.telephone = formulaire.value.telephone.trim()
    contact.email = formulaire.value.email.trim()
    idEnEdition.value = null
  } else {
    // Mode ajout
    contacts.value.push({
      id: Date.now(),
      prenom: formulaire.value.prenom.trim(),
      nom: formulaire.value.nom.trim(),
      telephone: formulaire.value.telephone.trim(),
      email: formulaire.value.email.trim(),
      supprimeLe: null // null = contact actif, pas dans la corbeille
    })
  }

  resetFormulaire()
}

function resetFormulaire() {
  formulaire.value = { prenom: '', nom: '', telephone: '', email: '' }
}

// --- Pré-remplir le formulaire pour modifier un contact existant ---
function modifierContact(contact) {
  idEnEdition.value = contact.id
  formulaire.value = {
    prenom: contact.prenom,
    nom: contact.nom,
    telephone: contact.telephone,
    email: contact.email || ''
  }
}

function annulerEdition() {
  idEnEdition.value = null
  resetFormulaire()
}

// --- Déplacer un contact vers la corbeille (pas de suppression immédiate) ---
function mettreALaCorbeille(id) {
  const contact = contacts.value.find(c => c.id === id)
  contact.supprimeLe = Date.now()
}

// --- Ressortir un contact de la corbeille ---
function restaurerContact(id) {
  const contact = contacts.value.find(c => c.id === id)
  contact.supprimeLe = null
}

// --- Contacts actifs (hors corbeille) ---
const contactsActifs = computed(() => {
  return contacts.value.filter(c => !c.supprimeLe)
})

// --- Contacts actifs triés alphabétiquement (nom, puis prénom) ---
const contactsTries = computed(() => {
  return [...contactsActifs.value].sort((a, b) => {
    const comparaisonNom = a.nom.localeCompare(b.nom, 'fr', { sensitivity: 'base' })
    if (comparaisonNom !== 0) return comparaisonNom
    return a.prenom.localeCompare(b.prenom, 'fr', { sensitivity: 'base' })
  })
})

// --- Contacts dans la corbeille, les plus récemment supprimés en premier ---
const contactsCorbeille = computed(() => {
  return contacts.value
    .filter(c => c.supprimeLe)
    .sort((a, b) => b.supprimeLe - a.supprimeLe)
})

// --- Jours restants avant suppression définitive ---
function joursRestants(contact) {
  const ecoule = Date.now() - contact.supprimeLe
  const restant = DUREE_CORBEILLE_MS - ecoule
  return Math.max(0, Math.ceil(restant / (24 * 60 * 60 * 1000)))
}

// --- Initiales pour l'avatar ---
function initiales(contact) {
  const i1 = contact.prenom.charAt(0).toUpperCase()
  const i2 = contact.nom.charAt(0).toUpperCase()
  return i1 + i2
}
</script>

<style scoped>
.carnet-app {
  --bg: #FFFFFF;
  --card: #e9760a86;
  --ink: #2E2A26;
  --muted: #8C8378;
  --accent: #da5011;
  --accent-soft: #e66611;
  --danger: #B85450;
  --success: #7A8C6E;
  --border: #a4f558;

  max-width: 520px;
  margin: 0 auto;
  padding: 48px 20px;
  color: var(--ink);
  font-family: 'Segoe UI', system-ui, -apple-system, sans-serif;
}

header { margin-bottom: 24px; }
header h1 { font-size: 28px; margin: 0 0 4px 0; letter-spacing: -0.5px; }
header p { margin: 0; color: var(--muted); font-size: 14px; }

.tabs {
  display: flex;
  gap: 6px;
  margin-bottom: 20px;
  border-bottom: 1px solid var(--border);
}

.tabs button {
  background: transparent;
  border: black;
  color: var(--muted);
  padding: 10px 4px;
  margin-right: 16px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  border-bottom: 2px solid transparent;
  transition: all 0.15s;
}

.tabs button.active {
  color: var(--accent);
  border-bottom-color: var(--accent);
}

.contact-form {
  background: var(--card);
  border: 1px solid var(--border);
  border-radius: 12px;
  padding: 16px;
  margin-bottom: 20px;
}

.form-row {
  display: flex;
  gap: 8px;
  margin-bottom: 8px;
}

.form-row input {
  flex: 1;
  padding: 10px 12px;
  border: 1px solid var(--border);
  border-radius: 8px;
  font-size: 14px;
  background: var(--bg);
  color: var(--ink);
  outline: none;
  transition: border-color 0.15s;
  min-width: 0;
}

.form-row input:focus { border-color: var(--accent); }

.form-actions {
  display: flex;
  gap: 8px;
  margin-top: 8px;
}

.form-actions button[type="submit"] {
  flex: 1;
  background: var(--accent);
  color: white;
  border: none;
  border-radius: 8px;
  padding: 10px 16px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  transition: opacity 0.15s;
}

.form-actions button[type="submit"]:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.btn-annuler {
  background: transparent;
  border: 1px solid var(--border);
  color: var(--muted);
  border-radius: 8px;
  padding: 10px 16px;
  font-size: 14px;
  cursor: pointer;
}

.contact-list {
  list-style: none;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.contact-item {
  display: flex;
  align-items: center;
  gap: 12px;
  background: var(--card);
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 12px 14px;
}

.contact-avatar {
  width: 38px;
  height: 38px;
  border-radius: 50%;
  background: var(--accent-soft);
  color: var(--accent);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 13px;
  font-weight: 700;
  flex-shrink: 0;
}

.contact-avatar.muted {
  background: var(--border);
  color: var(--muted);
}

.contact-infos {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.contact-nom { font-size: 15px; font-weight: 600; }
.contact-detail { font-size: 13px; color: var(--muted); }
.contact-detail.expiration { color: var(--danger); }

.contact-actions {
  display: flex;
  gap: 4px;
  flex-shrink: 0;
}

.icon-btn {
  background: none;
  border: none;
  color: var(--muted);
  font-size: 16px;
  cursor: pointer;
  padding: 6px;
  border-radius: 6px;
  transition: all 0.15s;
}

.icon-btn:hover { background: var(--accent-soft); color: var(--accent); }
.icon-btn.delete:hover { background: #F5E4E3; color: var(--danger); }
.icon-btn.restore:hover { background: #E9EEE5; color: var(--success); }

.empty-state {
  text-align: center;
  color: var(--muted);
  padding: 40px 0;
  font-size: 14px;
}
</style>