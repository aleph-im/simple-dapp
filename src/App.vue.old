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
          <b-link v-if="!account" @click="register">
            Register
          </b-link>
          <b-link v-if="!account" @click="login">
            Log-In
          </b-link>
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
const secp256k1 = require('secp256k1')
import {private_key_to_public_key,
  address_from_hash,
  public_key_to_hash,
  nuls_sign
} from './api/crypto.js'
import {fetch_profile, submit_aggregate} from './api/aggregates.js'
import {get_posts} from './api/posts.js'
import {broadcast, create_post} from './api/create.js'

var api_server = 'https://apitest.aleph.im'
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
      private_key: null,
      form_body: '',
      rooms: ['hall', 'troll', 'techies']
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
      let result = await get_posts('chat', {'refs': [this.room], 'api_server': api_server})
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
    },
    async edit_name() {
      let new_name = prompt("Please choose a name:", this.get_name(this.account.address))
      let message = await submit_aggregate(
        this.account.address, 'profile', {'name': new_name}, {api_server: api_server}
      )
      nuls_sign(Buffer.from(this.account.private_key, 'hex'), message)
      await broadcast(message, {api_server: api_server})
      await sleep(100)
      await this.fetch_profile(this.account.address)
    },
    check_pkey() {
      if (!isHex(this.private_key)) { return false }
      if (!this.private_key) { return false }
      if ((this.private_key.length === 66) && (this.private_key.substring(0, 2) === '00')) {
        this.private_key = this.private_key.substring(2, 66)
        return true
      }
      if (this.private_key.length !== 64) { return false }
      try {
        let prvbuffer = Buffer.from(this.private_key, 'hex')
        let pub = private_key_to_public_key(prvbuffer)
        return true
      } catch (e) {
        return false
      }
    },
    async register() {
      let randArr = new Uint8Array(32) // create a typed array of 32 bytes (256 bits)
      let privKey
      do {
        /// privKey = randomBytes(32)
        console.log('generating')
        window.crypto.getRandomValues(randArr)
        privKey = Buffer.from(randArr)
      } while (!secp256k1.privateKeyVerify(privKey))
      window.crypto.getRandomValues(randArr) // populate array with cryptographically secure random numbers
      alert("This is your private key, save it: " + privKey.toString('hex'))
      await this.add_account(privKey.toString('hex'))
    },
    async login() {
      this.private_key = prompt("Please enter your private key:")
      if (!this.check_pkey()) {
        alert("Private key is invalid.")
        return
      }
      await this.add_account(this.private_key)
    },
    async add_account(prv) {
      this.private_key = prv
      let prvbuffer = Buffer.from(this.private_key, 'hex')
      let pub = private_key_to_public_key(prvbuffer)
      let hash = public_key_to_hash(pub, {
        chain_id: this.chain_id
      })
      let address = address_from_hash(hash)
      // Vue.set(this, 'public_key', pub);
      let public_key = pub.toString('hex')
      let address_hash = hash.toString('hex')
      this.account = {
        'name': address,
        'private_key': this.private_key,
        'public_key': public_key,
        'address': address
      }
      await this.fetch_profile(address)
    },
    async logout() {
      this.account = null
    },
    write_message() {
      this.form_body = ''
      this.writeShow = true
    },
    async fetch_profile(address) {
      let profile = await fetch_profile(address, {api_server: api_server})
      this.profiles[address] = profile
    },
    async broadcast() {
      let msg = await create_post(
        this.account.address, 'chat',
        this.form_body, {
          ref: this.room,
          api_server: api_server
        }
      )
      nuls_sign(Buffer.from(this.account.private_key, 'hex'), msg)
      await broadcast(msg, {api_server: api_server})
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
