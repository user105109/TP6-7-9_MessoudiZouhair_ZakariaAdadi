<template>
  <p class="loading" v-show="loading">Chargement...</p>
  <p class="error" v-if="error">{{ error }}</p>

  <div v-if="country">
    <img
      :src="country.flags.svg"
      :alt="`Drapeau ${country.name.common}`"
      style="width: 200px;"
    />
    <h1>{{ country.name.common }}</h1>
    <p>Nom officiel : {{ country.name.official }}</p>
    <p>Capitale : {{ country.capital ? country.capital[0] : '—' }}</p>
    <p>Région : {{ country.region }}</p>
    <p>Population : {{ country.population.toLocaleString() }}</p>
  </div>

  <button @click="$router.back()">Retour</button>
</template>

<script>
export default {
  name: 'CountryDetailView',
  data() {
    return {
      country: null,
      loading: true,
      error: null
    }
  },
  methods: {
    async chargerPays() {
      this.loading = true
      this.error = null
      this.country = null

      try {
        const code = this.$route.params.code
        const url = `https://restcountries.com/v3.1/alpha/${encodeURIComponent(code)}?fields=name,capital,region,population,flags,cca3`

        const reponse = await fetch(url)

        if (!reponse.ok) {
          throw new Error(`Erreur HTTP : ${reponse.status}`)
        }

        const donnees = await reponse.json()

        // L'API peut renvoyer un objet ou un tableau contenant un objet.
        this.country = Array.isArray(donnees) ? donnees[0] : donnees

        if (!this.country) {
          throw new Error('Pays introuvable')
        }
      } catch (e) {
        console.error('Erreur lors du chargement du pays :', e)
        this.error = 'Impossible de charger les détails de ce pays.'
      } finally {
        this.loading = false
      }
    }
  },
  watch: {
    // Bonus : recharger la fiche si le paramètre code change
    '$route.params.code'() {
      this.chargerPays()
    }
  },
  mounted() {
    this.chargerPays()
  }
}
</script>
