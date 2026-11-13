<template>
  <div v-if="student">
    <h1>{{ student.firstName }} {{ student.lastName }}</h1>

    <p>
      Filière :
      <input v-model="student.filiere" type="text" />
    </p>

    <p>
      Année :
      <input v-model.number="student.year" type="number" min="1" />
    </p>

    <p>
      Email :
      <input v-model="student.email" type="email" />
    </p>
  </div>

  <div v-else>
    <p>Étudiant introuvable.</p>
  </div>

  <button @click="$router.back()">Retour</button>
  <router-link to="/">Retour à la liste</router-link>
</template>

<script>
import { store } from '../store/students.js'

export default {
  name: 'StudentDetailView',
  computed: {
    student() {
      // Le paramètre de route est une chaîne : on le convertit en nombre.
      const id = Number(this.$route.params.id)

      // find() renvoie l'objet original présent dans store.students.
      return store.students.find(student => student.id === id)
    }
  }
}
</script>
