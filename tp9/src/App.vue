<template>
  <div class="app">
    <header class="topbar">
      <h1>Météo en direct</h1>
      <button class="unit-toggle" @click="basculerUnite" v-if="ville">
        Afficher en {{ unite === 'metric' ? '°F' : '°C' }}
      </button>
    </header>

    <div class="search">
      <input
          v-model="query"
          @input="onInputRecherche"
          type="text"
          placeholder="Rechercher une ville... (ex: Safi, Paris, Tokyo)"
      />
      <p class="hint" v-if="chargementSuggestions">Recherche...</p>
      <p class="msg-error" v-if="erreurSuggestions">{{ erreurSuggestions }}</p>

      <ul class="suggestions" v-if="suggestions.length">
        <li v-for="s in suggestions" :key="s.lat + '-' + s.lon" @click="choisirVille(s)">
          {{ s.name }}<span v-if="s.state">, {{ s.state }}</span> ({{ s.country }})
        </li>
      </ul>
    </div>

    <div class="historique" v-if="historique.length">
      <span>Récent :</span>
      <button v-for="h in historique" :key="h.lat + '-' + h.lon" @click="choisirVille(h)">
        {{ h.name }}
      </button>
    </div>

    <div class="meteo" v-if="ville">
      <p class="loading" v-if="chargementMeteo">Chargement de la météo...</p>
      <p class="msg-error" v-else-if="erreurMeteo">{{ erreurMeteo }}</p>

      <div v-else-if="meteoActuelle" class="courante">
        <h2>{{ ville.name }}, {{ ville.country }}</h2>
        <div class="temp-actuelle">
          <img :src="iconeUrl(meteoActuelle.weather[0].icon)" :alt="meteoActuelle.weather[0].description" />
          <span class="valeur">{{ Math.round(meteoActuelle.main.temp) }}°</span>
          <span class="description">{{ meteoActuelle.weather[0].description }}</span>
        </div>
        <p class="details">
          Ressenti {{ Math.round(meteoActuelle.main.feels_like) }}° · Humidité {{ meteoActuelle.main.humidity }}%
        </p>
      </div>

      <div class="previsions" v-if="previsionsParJour.length">
        <div class="jour" v-for="j in previsionsParJour" :key="j.date">
          <span class="date">{{ j.date }}</span>
          <img :src="iconeUrl(j.icone)" :alt="j.description" />
          <span class="temps">{{ Math.round(j.min) }}° / {{ Math.round(j.max) }}°</span>
        </div>
      </div>
    </div>

    <p class="empty" v-if="!ville">Cherchez une ville pour voir sa météo.</p>
  </div>
</template>

<script>
const API_KEY = import.meta.env.VITE_OWM_API_KEY
const GEO_URL = 'https://api.openweathermap.org/geo/1.0/direct'
const WEATHER_URL = 'https://api.openweathermap.org/data/2.5/weather'
const FORECAST_URL = 'https://api.openweathermap.org/data/2.5/forecast'

export default {
  name: 'App',
  data() {
    return {
      query: '',
      debounceTimer: null,
      suggestions: [],
      chargementSuggestions: false,
      erreurSuggestions: null,

      ville: null,
      historique: [],

      meteoActuelle: null,
      previsions: [],
      chargementMeteo: false,
      erreurMeteo: null,

      unite: 'metric'
    }
  },
  computed: {
    previsionsParJour() {
      const jours = {}
      for (const p of this.previsions) {
        const [date, heure] = p.dt_txt.split(' ')
        const ecart = Math.abs(Number(heure.slice(0, 2)) - 12)
        const j = jours[date]
        if (!j) {
          jours[date] = {
            date,
            min: p.main.temp_min,
            max: p.main.temp_max,
            icone: p.weather[0].icon,
            description: p.weather[0].description,
            ecart
          }
        } else {
          j.min = Math.min(j.min, p.main.temp_min)
          j.max = Math.max(j.max, p.main.temp_max)
          if (ecart < j.ecart) {
            j.icone = p.weather[0].icon
            j.description = p.weather[0].description
            j.ecart = ecart
          }
        }
      }
      return Object.values(jours).sort((a, b) => a.date.localeCompare(b.date))
    }
  },
  methods: {
    iconeUrl(code) {
      return `https://openweathermap.org/img/wn/${code}@2x.png`
    },

    onInputRecherche() {
      clearTimeout(this.debounceTimer)
      if (!this.query.trim()) {
        this.suggestions = []
        this.erreurSuggestions = null
        return
      }
      this.debounceTimer = setTimeout(() => this.rechercherVilles(), 400)
    },

    async rechercherVilles() {
      this.chargementSuggestions = true
      this.erreurSuggestions = null
      try {
        const reponse = await fetch(
            `${GEO_URL}?q=${encodeURIComponent(this.query)}&limit=5&appid=${API_KEY}`
        )
        if (reponse.status === 401) {
          this.suggestions = []
          this.erreurSuggestions = 'Clé API invalide ou absente.'
          return
        }
        if (reponse.status === 429) {
          this.suggestions = []
          this.erreurSuggestions = 'Quota dépassé, réessayez plus tard.'
          return
        }
        if (!reponse.ok) {
          this.suggestions = []
          this.erreurSuggestions = 'Erreur serveur : ' + reponse.status
          return
        }
        const donnees = await reponse.json()
        this.suggestions = donnees
        if (donnees.length === 0) {
          this.erreurSuggestions = 'Aucune ville trouvée pour cette recherche.'
        }
      } catch (e) {
        this.suggestions = []
        this.erreurSuggestions = 'Erreur réseau, vérifiez votre connexion.'
      } finally {
        this.chargementSuggestions = false
      }
    },

    choisirVille(ville) {
      this.suggestions = []
      this.query = ''
      this.erreurSuggestions = null
      this.ville = ville
      this.historique = [
        ville,
        ...this.historique.filter((h) => !(h.lat === ville.lat && h.lon === ville.lon))
      ].slice(0, 5)
      this.chargerMeteo()
    },

    async chargerMeteo() {
      this.chargementMeteo = true
      this.erreurMeteo = null
      const { lat, lon } = this.ville
      const params = `lat=${lat}&lon=${lon}&appid=${API_KEY}&units=${this.unite}&lang=fr`
      try {
        const [repMeteo, repPrevisions] = await Promise.all([
          fetch(`${WEATHER_URL}?${params}`),
          fetch(`${FORECAST_URL}?${params}`)
        ])
        if (repMeteo.status === 401 || repPrevisions.status === 401) {
          throw new Error('Clé API invalide ou absente.')
        }
        if (repMeteo.status === 429 || repPrevisions.status === 429) {
          throw new Error('Quota dépassé, réessayez plus tard.')
        }
        if (!repMeteo.ok || !repPrevisions.ok) {
          throw new Error('Erreur serveur.')
        }
        const meteo = await repMeteo.json()
        const previsions = await repPrevisions.json()
        this.meteoActuelle = meteo
        this.previsions = previsions.list
      } catch (e) {
        this.meteoActuelle = null
        this.previsions = []
        if (e instanceof TypeError) {
          this.erreurMeteo = 'Erreur réseau, vérifiez votre connexion.'
        } else {
          this.erreurMeteo = e.message
        }
      } finally {
        this.chargementMeteo = false
      }
    },

    basculerUnite() {
      this.unite = this.unite === 'metric' ? 'imperial' : 'metric'
      this.chargerMeteo()
    }
  }
}
</script>