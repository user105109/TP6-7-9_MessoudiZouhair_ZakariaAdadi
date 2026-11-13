<template>
  <h1>Explorateur de pays</h1>

  <p class="loading" v-show="loading">Chargement des pays...</p>
  <p class="error" v-if="error">{{ error }}</p>

  <div class="toolbar">
    <input
      v-model.trim="recherche"
      type="search"
      placeholder="Rechercher un pays..."
    />
  </div>

  <div class="grid">
    <CountryCard
      v-for="country in paysAffiches"
      :key="country.cca3"
      :country="country"
    />
  </div>

  <p v-if="!loading && !error && paysAffiches.length === 0">
    Aucun pays ne correspond à votre recherche.
  </p>
</template>

<script>
import CountryCard from '../components/CountryCard.vue'

const API_URL = 'https://restcountries.com/v3.1/all?fields=name,capital,region,population,flags,cca3'

export default {
  name: 'CountryListView',
  components: { CountryCard },
  data() {
    return {
      countries: [],
      loading: true,
      error: null,
      recherche: ''
    }
  },
  computed: {
    paysAffiches() {
      const texte = this.recherche.trim().toLowerCase()

      return this.countries.filter(country =>
        country.name.common.toLowerCase().includes(texte)
      )
    }
  },
  methods: {
    async chargerPays() {
      this.loading = true
      this.error = null

      try {
        const reponse = await fetch(API_URL)

        // fetch ne considère pas automatiquement les réponses HTTP 4xx/5xx
        // comme des erreurs : il faut vérifier response.ok.
        if (!reponse.ok) {
          throw new Error(`Erreur HTTP : ${reponse.status}`)
        }

        this.countries = await reponse.json()
      } catch (e) {
        console.error('Erreur lors du chargement des pays :', e)
        this.error = 'Impossible de charger les pays. Vérifiez votre connexion et réessayez.'
      } finally {
        // Cette instruction s'exécute en cas de succès comme en cas d'erreur.
        this.loading = false
      }
    }
  },
  mounted() {
    this.chargerPays()
  }
}
</script>
