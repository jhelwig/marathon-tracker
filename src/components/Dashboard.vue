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
            <input type="text" name="clientId" v-model="apiConfiguration.clientId" />
          </label>
          <label name="clientSecret">StreamLabs Client Secret
            <input type="text" name="clientSecret" v-model="apiConfiguration.clientSecret" />
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
      apiConfiguration: this.parseConfig(),
      apiConfigurationErrors: [],
      hasConfig: this.hasValidConfiguration()
    }
  },
  created () {
    console.log('We are in the created hook')
    if (this.$route.query.code) {
      let apiConfig = this.parseConfig()
      apiConfig.oauthCode = this.$route.query.code
      localStorage.setItem('apiConfiguration', JSON.stringify(apiConfig))
      this.apiConfiguration.oauthCode = apiConfig.oauthCode
      console.log('We saved "code"')
    }

    if (this.apiConfiguration.oauthCode) {
      console.log('getting new socket token')
      this.getAccessToken()
    }
  },
  computed: {
    hasConfigErrors () {
      return this.apiConfigurationErrors.length
    },
    needsConfiguration () {
      return !this.hasConfig
    }
  },
  methods: {
    configurationSubmit (event) {
      event.preventDefault()
      const apiConfig = JSON.stringify({
        clientId: this.apiConfiguration.clientId,
        clientSecret: this.apiConfiguration.clientSecret
      })
      localStorage.setItem('apiConfiguration', apiConfig)
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

    getAccessToken () {
      const xhr = new XMLHttpRequest()
      // const data = JSON.stringify(false)
      xhr.withCredentials = true

      xhr.addEventListener('readystatechange', function () {
        console.log(arguments)
        if (this.readyState === this.DONE) {
          console.log('WE get this frm the access request')
          console.log(this.responseText)
        }
      })

      xhr.open('POST', `${this.apiBase}/token`)
      xhr.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded')

      xhr.send(`grant_type=authorization_code&client_id=${this.apiConfiguration.clientId}&client_secret=${this.apiConfiguration.clientSecret}&redirect_uri=${encodeURIComponent('http://lvh.me:8080/')}&code=${this.apiConfiguration.oauthCode}`)
    },

    getSocketToken () {
      const xhr = new XMLHttpRequest()
      const data = JSON.stringify(false)
      xhr.withCredentials = true

      xhr.addEventListener('readystatechange', function () {
        console.log(arguments)
        if (this.readyState === this.DONE) {
          console.log('WE get this frm the access request')
          console.log(this.responseText)
        }
      })

      xhr.open('GET', `${this.apiBase}/socket/token?access_token=${this.apiConfiguration.oauthCode}`)

      xhr.send(data)
    },

    hasValidConfiguration () {
      const apiConfig = this.parseConfig()
      return apiConfig.clientId && apiConfig.clientSecret
    },

    parseConfig () {
      return JSON.parse(localStorage.getItem('apiConfiguration')) || {}
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
