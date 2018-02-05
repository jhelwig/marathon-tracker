<template>
  <div>
    <div v-if="needsConfiguration">
      <ul v-if="hasConfigErrors">
        <li v-for="error in apiConfigurationErrors" :key="error.title">
          <h5>{{error.title}}</h5>
          <p>{{error.body}}</p>
        </li>
      </ul>
      <form v-on:submit="configurationSubmit">
        <fieldset>
          <label name="clientId">StreamLabs Client ID
            <input type="text" name="clientId" v-model="apiConfigurationForm.clientId" />
          </label>
          <label name="clientSecret">StreamLabs Client Secret
            <input type="text" name="clientSecret" v-model="apiConfigurationForm.clientSecret" />
          </label>
        </fieldset>
        <button type="submit">Save</button>
      </form>
    </div>
  </div>
</template>

<script>
import config from '../../config/index'

export default {
  name: 'Dashboard',
  data () {
    return {
      apiBase: config.dev.streamlabsApiBase,
      apiConfigurationForm: {},
      apiConfigurationErrors: []
    }
  },
  created () {
    console.log('We are in the created hook')
    if (this.$route.query.code) {
      const oauthCode = this.$route.query.code
      this.getAccessToken(oauthCode)
    }

    if (!this.apiConfiguration.socketToken && this.apiConfiguration.tokenDetails && this.apiConfiguration.tokenDetails.access_token) {
      this.getSocketToken()
    }

    if (this.apiConfiguration.socketToken) {
      this.startEventSocketStream()
    }
  },
  computed: {
    apiConfiguration () {
      console.log('Wr are pretending to compute', localStorage.getItem('apiConfiguration'))
      return JSON.parse(localStorage.getItem('apiConfiguration')) || {}
    },
    hasConfigErrors () {
      return this.apiConfigurationErrors.length
    },
    needsConfiguration () {
      return !this.hasValidConfiguration()
    }
  },
  methods: {
    configurationSubmit (event) {
      event.preventDefault()
      this.setApiConfigurationValue('clientId', this.apiConfigurationForm.clientId)
      this.setApiConfigurationValue('clientSecret', this.apiConfigurationForm.clientSecret)
      if (this.hasValidConfiguration()) {
        this.hasConfig = true
        window.location = `${this.apiBase}/authorize?client_id=${this.apiConfiguration.clientId}&redirect_uri=http://lvh.me:8080/&response_type=code&scope=donations.read+socket.token+points.read`
      } else {
        this.apiConfigurationErrors = [{
          title: 'Invalid Configuration',
          body: 'You are missing some api config thing'
        }]
      }
    },

    getAccessToken (oauthCode) {
      const xhr = new XMLHttpRequest()
      // const data = JSON.stringify(false)
      xhr.withCredentials = true

      xhr.addEventListener('load', (event) => {
        this.setApiConfigurationValue('tokenDetails', JSON.parse(event.currentTarget.response))
        this.$router.replace('/')
      })

      xhr.open('POST', `${this.apiBase}/token`)
      xhr.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded')

      xhr.send(`grant_type=authorization_code&client_id=${this.apiConfiguration.clientId}&client_secret=${this.apiConfiguration.clientSecret}&redirect_uri=${encodeURIComponent('http://lvh.me:8080/')}&code=${oauthCode}`)
    },

    getSocketToken () {
      const xhr = new XMLHttpRequest()
      const data = JSON.stringify(false)
      xhr.withCredentials = true

      xhr.addEventListener('load', (event) => {
        this.setApiConfigurationValue('socketToken', JSON.parse(event.currentTarget.response).socket_token)
        this.$router.replace('/')
      })

      xhr.open('GET', `${this.apiBase}/socket/token?access_token=${this.apiConfiguration.tokenDetails.access_token}`)
      xhr.send(data)
    },

    hasValidConfiguration () {
      return this.apiConfiguration.clientId && this.apiConfiguration.clientSecret
    },

    setApiConfigurationValue (name, value) {
      let apiConfig = this.apiConfiguration
      apiConfig[name] = value
      localStorage.setItem('apiConfiguration', JSON.stringify(apiConfig))
    },

    startEventSocketStream () {
      const streamlabs = io(`https://sockets.streamlabs.com?token=${this.apiConfiguration.socketToken}`)

      streamlabs.on('event', this.handleSocketEvent.bind(this))
    },

    handleSocketEvent (eventData) {
      console.log(eventData)
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h1, h2 {
  font-weight: normal;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
label {
  display: block;
}
</style>
