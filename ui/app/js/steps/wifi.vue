<template>
  <span v-if="!loading">
    <p>
      Sélectionnez le réseau WiFi auquel Ingosi doit se connecter :
    </p>

    <form @submit.prevent="sendDone">
      <label class="label" for="wifi_network">Réseau</label>
      <p class="control">
        <span class="select">
          <select v-model="ssid_select" id="wifi_network">
            <option value="select">Sélectionner...</option>
            <option v-for="network in formattedNetworks" :value="network">
              {{ network.ssid }} - {{ network.encryption }} ({{ network.signalQuality }}%)
            </option>
            <option value="other">Réseau caché</option>
          </select>
        </span>
      </p>

      <div v-if="ssid_select === 'other'">
        <label class="label" for="wifi_ssid">Wi-Fi SSID</label>
        <p class="control">
          <input class="input" v-model.trim="ssid_input" id="wifi_ssid" type="text" placeholder="SSID" maxLength="32" required />
          <span class="help">Requis.</span>
        </p>
      </div>

      <div v-if="typeof ssid_select === 'object' && ssid_select.encryption !== 'Open' || ssid_select === 'other'">
        <label class="label" for="wifi_password">Mot de passe WiFi</label>
        <p class="control is-grouped">
          <!-- v-model does not support dynamic :type -->
          <input v-if="passwordClearText" type="text" class="input" v-model.trim="password" id="wifi_password" placeholder="Mot de passe" maxLength="64" />
          <input v-else type="password" class="input" v-model.trim="password" id="wifi_password" placeholder="Mot de passe" maxLength="64" />
          <label class="checkbox">
            <input type="checkbox" v-model="passwordClearText" /> Afficher le mot de passe
          </label>
        </p>
        <span class="help">Laisser vide s'il n'y a pas de mot de passe WiFi</span>
        <br/>
      </div>

      <p class="control">
        <button type="submit" :disabled="!formIsValid" class="button is-primary">Suivant</button>
      </p>
    </form>
  </span>
</template>

<script>
import axios from 'axios'
import {API_URL} from '../constants'

export default {
  data () {
    return {
      loading: true,
      apiResponse: null,
      ssid_select: 'select',
      ssid_input: null,
      password: null,
      passwordClearText: false
    }
  },
  computed: {
    formattedNetworks: function () {
      let networks = this.apiResponse.networks.slice(0)
      networks.sort(function (networkA, networkB) {
        if (networkA.rssi > networkB.rssi) {
          return -1
        } else if (networkA.rssi < networkB.rssi) {
          return 1
        } else {
          return 0
        }
      })

      networks = networks.map(function (network) {
        if (network.rssi <= -100) {
          network.signalQuality = 0
        } else if (network.rssi >= -50) {
          network.signalQuality = 100
        } else {
          network.signalQuality = 2 * (network.rssi + 100)
        }

        switch (network.encryption) {
          case 'wep':
            network.encryption = 'WEP'
            break
          case 'wpa':
            network.encryption = 'WPA'
            break
          case 'wpa2':
            network.encryption = 'WPA2'
            break
          case 'none':
            network.encryption = 'Open'
            break
          case 'auto':
            network.encryption = 'Automatic'
            break
        }
        return network
      })

      return networks
    },
    computedSsid: function () {
      let ssid;
      if (typeof this.ssid_select === 'object') return this.ssid_select.ssid
      else if (this.ssid_select === 'other') return this.ssid_input

      return null
    },
    formIsValid: function () {
      const needPassword = typeof this.ssid_select === 'object' && this.ssid_select.encryption !== 'Open'

      return this.computedSsid && (!needPassword || this.password)
    }
  },
  mounted () {
    this.$emit('loading', 'Gathering available Wi-Fi networks...')
    axios.get(`${API_URL}/networks`).then((res) => {
      this.$emit('loaded')
      this.apiResponse = res.data
      this.loading = false
    }).catch((err) => {
    })
  },
  methods: {
    sendDone: function () {
      this.$emit('wifiConfig', {
        ssid: this.computedSsid,
        password: typeof this.ssid_select === 'object' && this.ssid_select.encryption === 'Open' ? null : this.password
      })

      this.$emit('done')
    }
  }
}
</script>
