<template>
  <modal-dialog v-ref:dialog bs-style="modal-lg">
    <div class="modal-header">
      <button type="button" class="close" @click="toggle" aria-label="Close">
        <span aria-hidden="true">×</span>
      </button>
      <h4 class="modal-title">New project</h4>
    </div>
    <div class="modal-body">
      <div class="form-group">
        <label>Name</label>
        <input type="text" class="form-control" placeholder="" v-model="name">
      </div>
      <div class="form-group">
        <label>Directory</label>
        <div class="input-group">
          <input type="text" class="form-control" placeholder="" v-model="directory">
          <span class="input-group-btn">
            <button class="btn btn-primary" @click="showOpenDialog">Choose…</button>
          </span>
        </div>
      </div>
      <div class="form-group">
        <label>Region</label>
        <select class="form-control" v-model="region">
          <option v-for="region in availableRegions" :value="region">{{region}}</option>
        </select>
      </div>
      <div class="form-group">
        <label>API</label>
        <div class="input-group">
          <select class="form-control" v-model="api">
            <option v-for="api in apis" :value="api">{{api.name}}</option>
          </select>
          <span class="input-group-btn">
            <button class="btn btn-primary" @click="$refs.modalApi.toggle()">Create…</button>
          </span>
        </div>
      </div>
    </div>
    <div class="modal-footer">
      <button type="button" class="btn btn-default" @click="toggle">Close</button>
      <button type="button" class="btn btn-primary" @click="init">Create</button>
    </div>
  </modal-dialog>

  <modal-api v-ref:modal-api @create-api="onCreateApi"></modal-api>
</template>

<script>
var electron = require('electron')

import Vue from 'vue'

import ModalDialog from './ModalDialog.vue'
import ModalAPI from './ModalApi.vue'

import MixinState from '../mixins/mixin-state'

import backbeam from '../utils/backbeam-singleton'
import subscriber from '../utils/backbeam-subscriber'
import errorHandler from '../utils/error-handler'

export default Vue.component('modal-project-init', {
  mixins: [MixinState({
    apis: [],
    api: null,
    name: '',
    directory: '',
    region: 'us-east-1',
    availableRegions: ['us-east-1', 'us-west-2', 'eu-west-1', 'ap-northeast-1'],
  })],
  created() {
    this.subscriber = subscriber(this, 'credentials_changed')
    if (backbeam.credentialsLoaded) {
      this.refreshApis()
    }
  },
  destroyed() {
    this.subscriber.unsubscribe()
  },
  methods: {
    onCredentialsChanged() {
      this.refreshApis()
    },
    refreshApis() {
      backbeam.apiList()
        .then(list => {
          this.apis = list.items
          this.api = list.items[0]
        })
        .catch(errorHandler)
    },
    onRegionChange(region) {
      backbeam.setRegion(region)
      this.region = region
      this.apis = []
      this.refreshApis()
    },
    onCreateApi(data) {
      backbeam.apiCreate(data)
        .then(api => {
          this.api = api
          this.apis.push(api)
        })
        .catch(errorHandler)
    },
    showOpenDialog() {
      const dialog = electron.remote.dialog
      const dir = dialog.showOpenDialog(electron.remote.getCurrentWindow(), {
        properties: [ 'openDirectory', 'createDirectory' ],
      })
      if (!dir || dir.length === 0) return
      this.directory = dir[0]
    },
    toggle() {
      this.$refs.dialog.toggle()
    },
    init() {
      const dir = this.directory
      const params = this.pick('api', 'name', 'region')
      backbeam.init(dir, params)
        .then(() => {
          this.toggle()
          sweetAlert('Success', 'Run `npm install` to install dependencies', 'success')
        })
        .catch(errorHandler)
    }
  }
})
</script>
