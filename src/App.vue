<template>
  <div id="app">
    <notifications group="wallet" />
    <b-modal id="writeModal" ref="writeModal" hide-footer title="Write a review" v-model="writeShow">
      <b-form-group
      id="fieldset1"
      label="Your message"
      label-for="form_body"
      >
        <b-form-textarea id="form_body" v-model="form_body"></b-form-textarea>
      </b-form-group>

      <button class="btn btn-lg btn-block btn-primary mb-3" v-if="form_body" v-on:click="broadcast">
        Submit
      </button>
    </b-modal>
    <b-container>
      <div class="d-flex justify-content-between">
        <h1>Aleph example dApp</h1>
        <div>
          <div v-if="!account" class="d-inline-block">
            NULS
            <p class="badge">
              <b-link @click="nuls_register" class="mr-2">Register</b-link>
              <b-link @click="nuls_login">Log-In</b-link>
            </p>
            ETH
            <p class="badge">
              <b-link @click="eth_register" class="mr-2">Register</b-link> 
              <b-link @click="eth_login" class="mr-2">Log-In</b-link> 
              <b-link @click="eth_web3_login">Web3</b-link>
            </p>
            NEO
            <p class="badge">
              <b-link @click="neo_register" class="mr-2">Register</b-link> 
              <b-link @click="neo_login" class="mr-2">Log-In</b-link>
            </p>
          </div>
          <p class="badge" v-else>
            Welcome {{get_name(account.address)}}! <b-link v-b-tooltip.hover title="Edit name" @click="edit_name">📝</b-link>
            <b-link @click="logout">
              Log-Out
            </b-link>
          </p>
          <b-link v-b-tooltip.hover title="Refresh" @click="refresh">🔄</b-link>
        </div>
      </div>
      <b-row class="mt-4">
        <b-col sm="12" md="4" lg="2">
          <h4>Rooms</h4>
          <b-list-group>
            <b-list-group-item v-for="rname of rooms" :key="rname">
              <b-link @click="room=rname">
                {{rname}}
              </b-link>
            </b-list-group-item>
          </b-list-group>
        </b-col>
        <b-col sm="12" md="8" lg="10">
          <div class="d-flex justify-content-between">
            <h2>{{room}}</h2>
            <b-button v-if="account" @click="write_message()" class="mb-4">
              Write a message
            </b-button>
          </div>
          <b-card v-for="post of posts" class="mb-2" :key="post.hash"
            :title="`Message from ${get_name(post.address)}`">
            {{post.content.body}}
          </b-card>
          <div class="d-flex flex-row-reverse">
            <b-button v-if="account" @click="write_message()" class="mt-4">
              Write a message
            </b-button>
          </div>
        </b-col>
      </b-row>
    </b-container>
  </div>
</template>

<script>
import {aggregates, posts, store, nuls2, ethereum, neo} from 'aleph-js'
console.log(ethereum)

var api_server = 'https://api2.aleph.im'
var network_id = 261

var hexRegEx = /([0-9]|[a-f])/gim

function isHex (input) {
  return typeof input === 'string' &&
    (input.match(hexRegEx) || []).length === input.length
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

export default {
  name: 'app',
  data () {
    return {
      msg: 'Welcome to Your Vue.js App',
      posts: [],
      room: 'hall',
      profiles: [],
      writeShow: false,
      account: null,
      chain_id: 261,
      mnemonics: null,
      wif: null,
      form_body: '',
      rooms: ['hall', 'troll', 'techies'],
      channel: 'TEST'
    }
  },
  watch: {
    async room() {
      this.messages = []
      await this.get_messages()
    }
  },
  methods: {
    async refresh() {
      await this.get_messages()
    },
    async get_messages() {
      let result = await posts.get_posts('chat', {'refs': [this.room], 'api_server': api_server})
      this.posts = result.posts
      for (let post of this.posts) {
        if (this.profiles[post.address] === undefined)
          await this.fetch_profile(post.address)
      }
    },
    get_name(address) {
      if ((this.profiles[address]) && (this.profiles[address].name))
        return this.profiles[address].name
      else
        return address
    }, //aa90f6ea5df5cbf50f72286058f7bf134bb0cbfb7fb5bb6495ebd1490dbc3a89
    async edit_name() {
      let new_name = prompt("Please choose a name:", this.get_name(this.account.address))
      let message = await aggregates.submit(
        this.account.address, 'profile',
        {'name': new_name},
        {
          account: this.account,
          api_server: api_server,
          channel: this.channel
        }
      )
      console.log(message)
      await sleep(100)
      await this.fetch_profile(this.account.address)
    },
    async nuls_register() {
      let account = await nuls2.new_account()
      alert("This is are your mnemonics, save them: " + account.mnemonics)
      this.account = account
    },
    async nuls_login() {
      this.mnemonics = prompt("Please enter your mnemonics:")
      let account = await this.add_account('NULS2', this.mnemonics)
      if (!account) {
        alert("Private key is invalid.")
        return
      }
    },
    async eth_register() {
      let account = await ethereum.new_account()
      alert("This is are your mnemonics, save them: " + account.mnemonics)
      this.account = account
    },
    async eth_login() {
      this.mnemonics = prompt("Please enter your mnemonics:")
      let account = await this.add_account('ETH', this.mnemonics)
      if (!account) {
        alert("Private key is invalid.")
        return
      }
    },
    async neo_register() {
      let account = await neo.new_account()
      alert("This is your private key (WIF), save them: " + account.wif)
      this.account = account
    },
    async neo_login() {
      this.wif = prompt("Please enter your private key (WIF):")
      let account = await this.add_account('NEO', this.wif)
      if (!account) {
        alert("Private key is invalid.")
        return
      }
    },
    async eth_web3_login() {
      let account = null
      if (window.ethereum) {
        try {
            // Request account access if needed
            await window.ethereum.enable()
            account = await ethereum.from_provider(window['ethereum'] || window.web3.currentProvider)
        } catch (error) {
            // User denied account access...
        }
      } else {
        alert('not supported?')
      }
      console.log(account)
      if (!account) {
        alert("Error getting web3 account")
        return
      }
      this.account = account
    },
    async add_account(type, word) {
      // this.mnemonics = mnemonics
      let account = null
      if (type == 'NULS2')
        account = await nuls2.import_account({mnemonics: word})
      else if (type == 'ETH')
        account = await ethereum.import_account({mnemonics: word})
      else if (type == 'NEO')
        account = await neo.import_account({wif: word})
      if (account) {
        this.account = account
        await this.fetch_profile(account.address)
      }
      return account
    },
    async logout() {
      this.account = null
    },
    write_message() {
      this.form_body = ''
      this.writeShow = true
    },
    async fetch_profile(address) {
      let profile = await aggregates.fetch_profile(address, {api_server: api_server})
      console.log(profile)
      this.profiles[address] = profile
    },
    async broadcast() {
      let msg = await posts.submit(
        this.account.address, 'chat',
        {'body': this.form_body}, {
          ref: this.room,
          api_server: api_server,
          account: this.account,
          channel: this.channel
        }
      )
      // nuls_sign(Buffer.from(this.account.private_key, 'hex'), msg)
      // await broadcast(msg, {api_server: api_server})
      await sleep(100)
      await this.refresh()
      this.form_body = ''
      this.writeShow = false
    }
  },
  async mounted() {
    console.log('blah')
    await this.refresh()
    setInterval(this.refresh, 1000);
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  margin-top: 60px;
}

h1, h2 {
  font-weight: normal;
  text-align: center;
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
</style>
