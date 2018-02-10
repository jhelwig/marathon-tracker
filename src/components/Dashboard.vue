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
    <div v-if="!needsConfiguration">
      <p>
        Total Donations: ${{ totalDonations }}<br />
        Total Bits: {{ totalBits }}<br />
        Sub Points: {{ subPoints }}<br />
        Resub Months: {{ resubMonths }}<br />
      </p>
      <p>
        Doraemon GB any% (glitched): {{ doraemonRuns }} ({{ donationDoraemonRuns }} + {{ bitsDoraemonRuns }})<br />
        TGM1 Minutes: {{ tgmMinutes }}<br />
      </p>
    </div>
  </div>
</template>

<script>
import config from '../../config/index'
// http://dexie.org/
import Dexie from 'dexie'

export default {
  name: 'Dashboard',
  data () {
    return {
      apiBase: config.dev.streamlabsApiBase,
      apiConfigurationForm: {},
      apiConfigurationErrors: [],
      totalDonations: 0,
      totalBits: 0,
      subPoints: 0,
      resubMonths: 0,
      doraemonRuns: 0,
      donationDoraemonRuns: 0,
      bitsDoraemonRuns: 0,
      tgmMinutes: 60
    }
  },
  watch: {
    totalDonations: function (newTotalDonations, oldTotalDonations) {
      let doraemonRuns = Math.floor(newTotalDonations / 10)
      this.donationDoraemonRuns = doraemonRuns
    },
    totalBits: function (newTotalBits, oldTotalBits) {
      let doraemonRuns = Math.floor(newTotalBits / 100)
      this.bitsDoraemonRuns = doraemonRuns
    }
  },
  created () {
    // this.db.delete()
    this.db.version(1).stores({
      donations: '_id,name,amount,message,currency,id'
    })
    this.db.version(2).stores({
      bits: '_id,name,amount,message,currency'
    })
    this.db.version(3).stores({
      subscriptions: '_id,name,sub_plan,months,message'
    })

    this.db.open().then(function (db) {
      // Database opened successfully
    }).catch(function (err) {
      console.log(err)
    })

    this.configureApi()

    if (this.hasValidConfiguration()) {
      this.db.donations.toCollection().toArray((donations) => {
        this.totalDonations = donations.reduce((a, b) => a + b.amount, 0)
      })
      this.db.bits.toCollection().toArray((bits) => {
        this.totalBits = bits.reduce((a, b) => a + b.amount, 0)
      })
      this.updateSubPointsMonths()
    }
    this.$watch(
      function () {
        return this.donationDoraemonRuns + this.bitsDoraemonRuns
      },
      function (newRunTotal, oldRunTotal) {
        if (newRunTotal > 100) {
          this.doraemonRuns = 100
        } else {
          this.doraemonRuns = newRunTotal
        }
      }
    )
    this.$watch(
      function () {
        return 60 + this.subPoints + this.resubMonths
      },
      function (newTgmMinutes, oldTgmMinutes) {
        if (newTgmMinutes > 240) {
          this.tgmMinutes = 240
        } else {
          this.tgmMinutes = newTgmMinutes
        }
      }
    )
  },
  computed: {
    apiConfiguration () {
      return JSON.parse(localStorage.getItem('apiConfiguration')) || {}
    },
    hasConfigErrors () {
      return this.apiConfigurationErrors.length
    },
    needsConfiguration () {
      return !this.hasValidConfiguration()
    },
    db () {
      return new Dexie('marathon_database')
    }
  },
  methods: {
    configureApi () {
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
        this.configureApi()
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
        this.configureApi()
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
      if (eventData.type === 'donation') {
        // donations: '_id,name,amount,message,currency,id'
        eventData.message.forEach((donationDetails) => {
          this.db.donations.put({
            _id: donationDetails._id,
            name: donationDetails.name,
            amount: Number(donationDetails.amount),
            message: donationDetails.message,
            currency: donationDetails.currency,
            id: donationDetails.id
          }).then(() => {
            this.db.donations.toCollection().toArray((donations) => {
              this.totalDonations = donations.reduce((a, b) => a + b.amount, 0)
            })
          }).catch(function (error) {
            console.log(error)
          })
        })
      } else if (eventData.type === 'bits') {
        // bits: '_id,name,amount,message,currency'
        eventData.message.forEach((bitsDetails) => {
          this.db.bits.put({
            _id: bitsDetails._id,
            name: bitsDetails.name,
            amount: Number(bitsDetails.amount),
            message: bitsDetails.message,
            currency: bitsDetails.currency
          }).then(() => {
            this.db.bits.toCollection().toArray((bits) => {
              this.totalBits = bits.reduce((a, b) => a + b.amount, 0)
            })
          }).catch(function (error) {
            console.log(error)
          })
        })
      } else if (eventData.type === 'subscription') {
        // subscriptions: '_id,name,sub_plan,months,message'
        eventData.message.forEach((subscriptionDetails) => {
          this.db.subscriptions.put({
            _id: subscriptionDetails._id,
            name: subscriptionDetails.name,
            sub_plan: subscriptionDetails.sub_plan,
            months: subscriptionDetails.months,
            message: subscriptionDetails.message
          }).then(this.updateSubPointsMonths.bind(this))
        })
      }
    },

    updateSubPointsMonths () {
      this.db.subscriptions.toCollection().toArray((subscriptions) => {
        let newResubMonths = 0
        let newSubPoints = 0

        subscriptions.forEach(function (subscription) {
          if (subscription.months > 1) {
            newResubMonths += subscription.months
          }
          switch (subscription.sub_plan) {
            case 'Prime':
            case '1000':
              newSubPoints += 1
              break
            case '2000':
              newSubPoints += 2
              break
            case '3000':
              newSubPoints += 3
              break
            default:
              console.log('Unknown sub_plan:', subscription.sub_plan)
          }
        })
        this.resubMonths = newResubMonths
        this.subPoints = newSubPoints
      })
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
