<template>
  <h1>Annuaire des étudiants</h1>

  <!-- Formulaire d'ajout -->
  <form @submit.prevent="ajouterEtudiant">
    <input
      v-model.trim="newStudent.firstName"
      type="text"
      placeholder="Prénom"
      required
    />
    <input
      v-model.trim="newStudent.lastName"
      type="text"
      placeholder="Nom"
      required
    />
    <input
      v-model.trim="newStudent.filiere"
      type="text"
      placeholder="Filière"
    />
    <input
      v-model.number="newStudent.year"
      type="number"
      min="1"
      placeholder="Année"
    />
    <input
      v-model.trim="newStudent.email"
      type="email"
      placeholder="Email"
    />
    <button type="submit">Ajouter</button>
  </form>

  <!-- Recherche et tri -->
  <div class="toolbar">
    <input
      v-model.trim="recherche"
      type="search"
      placeholder="Rechercher par prénom ou nom..."
    />

    <select v-model="tri">
      <option value="lastName">Trier par nom</option>
      <option value="filiere">Trier par filière</option>
    </select>
  </div>

  <!-- Résultat filtré et trié -->
  <p v-if="etudiantsAffiches.length === 0">
    Aucun étudiant trouvé.
  </p>

  <StudentCard
    v-for="student in etudiantsAffiches"
    :key="student.id"
    :student="student"
  />
</template>

<script>
import StudentCard from '../components/StudentCard.vue'
import { store } from '../store/students.js'

export default {
  name: 'StudentListView',
  components: { StudentCard },
  data() {
    return {
      store,
      recherche: '',
      tri: 'lastName',
      newStudent: {
        firstName: '',
        lastName: '',
        filiere: '',
        year: 1,
        email: ''
      }
    }
  },
  computed: {
    etudiantsAffiches() {
      const texte = this.recherche.toLowerCase()

      // 1. Filtrer par prénom ou nom
      const resultat = this.store.students.filter((student) => {
        const prenom = student.firstName.toLowerCase()
        const nom = student.lastName.toLowerCase()

        return prenom.includes(texte) || nom.includes(texte)
      })

      // 2. Trier le résultat selon le choix de l'utilisateur
      resultat.sort((a, b) => {
        return String(a[this.tri] || '').localeCompare(String(b[this.tri] || ''), 'fr')
      })

      return resultat
    }
  },
  methods: {
    ajouterEtudiant() {
      const firstName = this.newStudent.firstName.trim()
      const lastName = this.newStudent.lastName.trim()

      // Validation : prénom et nom obligatoires
      if (!firstName || !lastName) {
        alert('Veuillez renseigner le prénom et le nom.')
        return
      }

      // Générer un nouvel identifiant
      const nouvelId = this.store.students.length > 0
        ? Math.max(...this.store.students.map(student => student.id)) + 1
        : 1

      this.store.students.push({
        id: nouvelId,
        firstName,
        lastName,
        filiere: this.newStudent.filiere.trim(),
        year: Number(this.newStudent.year) || 1,
        email: this.newStudent.email.trim()
      })

      // Réinitialiser le formulaire
      this.newStudent = {
        firstName: '',
        lastName: '',
        filiere: '',
        year: 1,
        email: ''
      }
    }
  }
}
</script>
