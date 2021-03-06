<template>
  <div v-if="!loading">
    <div id="breadcrumbs">
      <router-link :to="'/share/' + hash" :aria-label="$t('files.home')" :title="$t('files.home')">
        <i class="material-icons">home</i>
      </router-link>

      <span v-for="(link, index) in breadcrumbs" :key="index">
          <span class="chevron"><i class="material-icons">keyboard_arrow_right</i></span>
          <router-link :to="link.url">{{ link.name }}</router-link>
        </span>
    </div>
    <div class="share">
      <div class="share__box share__box__info">
          <div class="share__box__header">
            {{ req.isDir ? $t('download.downloadFolder') : $t('download.downloadFile') }}
          </div>
          <div class="share__box__element share__box__center share__box__icon">
            <i class="material-icons">{{ icon }}</i>
          </div>
          <div class="share__box__element">
            <strong>{{ $t('prompts.displayName') }}</strong> {{ req.name }}
          </div>
          <div class="share__box__element">
            <strong>{{ $t('prompts.lastModified') }}:</strong> {{ humanTime }}
          </div>
          <div class="share__box__element">
            <strong>{{ $t('prompts.size') }}:</strong> {{ humanSize }}
          </div>
          <div class="share__box__element share__box__center">
            <a target="_blank" :href="link" class="button button--flat">{{ $t('buttons.download') }}</a>
          </div>
          <div class="share__box__element share__box__center">
            <qrcode-vue :value="fullLink" size="200" level="M"></qrcode-vue>
          </div>
      </div>
      <div v-if="req.isDir && req.items.length > 0" class="share__box share__box__items">
        <div class="share__box__header" v-if="req.isDir">
          {{ $t('files.files') }}
        </div>
        <div id="listing" class="list">
          <item v-for="(item) in req.items.slice(0, this.showLimit)"
            :key="base64(item.name)"
            v-bind:index="item.index"
            v-bind:name="item.name"
            v-bind:isDir="item.isDir"
            v-bind:url="item.url"
            v-bind:modified="item.modified"
            v-bind:type="item.type"
            v-bind:size="item.size">
          </item>
          <div v-if="req.items.length > showLimit" class="item">
            <div>
              <p class="name"> + {{ req.items.length - showLimit }} </p>
            </div>
          </div>

          <div :class="{ active: $store.state.multiple }" id="multiple-selection">
            <p>{{ $t('files.multipleSelectionEnabled') }}</p>
            <div @click="$store.commit('multiple', false)" tabindex="0" role="button" :title="$t('files.clear')" :aria-label="$t('files.clear')" class="action">
              <i class="material-icons">clear</i>
            </div>
          </div>
        </div>
      </div>
      <div v-else-if="req.isDir && req.items.length === 0" class="share__box share__box__items">
        <h2 class="message">
          <i class="material-icons">sentiment_dissatisfied</i>
          <span>{{ $t('files.lonely') }}</span>
        </h2>
      </div>
    </div>
  </div>
  <div v-else-if="error">
    <not-found v-if="error.message === '404'"></not-found>
    <forbidden v-else-if="error.message === '403'"></forbidden>
    <internal-error v-else></internal-error>
  </div>
</template>

<script>
import {mapState, mapMutations, mapGetters} from 'vuex';
import { share as api } from '@/api'
import { baseURL } from '@/utils/constants'
import filesize from 'filesize'
import moment from 'moment'
import QrcodeVue from 'qrcode.vue'
import Item from "@/components/files/ListingItem"
import Forbidden from './errors/403'
import NotFound from './errors/404'
import InternalError from './errors/500'

export default {
  name: 'share',
  components: {
    Item,
    Forbidden,
    NotFound,
    InternalError,
    QrcodeVue
  },
  data: () => ({
    error: null,
    path: '',
    showLimit: 500
  }),
  watch: {
    '$route': 'fetchData'
  },
  created: async function () {
    const hash = this.$route.params.pathMatch.split('/')[0]
    this.setHash(hash)
    await this.fetchData()
  },
  mounted () {
    window.addEventListener('keydown', this.keyEvent)
  },
  beforeDestroy () {
    window.removeEventListener('keydown', this.keyEvent)
  },
  computed: {
    ...mapState(['hash', 'req', 'loading', 'multiple']),
    ...mapGetters(['selectedCount']),
    icon: function () {
      if (this.req.isDir) return 'folder'
      if (this.req.type === 'image') return 'insert_photo'
      if (this.req.type === 'audio') return 'volume_up'
      if (this.req.type === 'video') return 'movie'
      return 'insert_drive_file'
    },
    link: function () {
      return `${baseURL}/api/public/dl/${this.hash}${this.path}`
    },
    fullLink: function () {
      return window.location.origin + this.link
    },
    humanSize: function () {
      if (this.req.isDir) {
        return this.req.items.length
      }

      return filesize(this.req.size)
    },
    humanTime: function () {
      return moment(this.req.modified).fromNow()
    },
    breadcrumbs () {
      let parts = this.path.split('/')

      if (parts[0] === '') {
        parts.shift()
      }

      if (parts[parts.length - 1] === '') {
        parts.pop()
      }

      let breadcrumbs = []

      for (let i = 0; i < parts.length; i++) {
        if (i === 0) {
          breadcrumbs.push({ name: decodeURIComponent(parts[i]), url: '/share/' + this.hash + '/' + parts[i] + '/' })
        } else  {
          breadcrumbs.push({ name: decodeURIComponent(parts[i]), url: breadcrumbs[i - 1].url + parts[i] + '/' })
        }
      }

      if (breadcrumbs.length > 3) {
        while (breadcrumbs.length !== 4) {
          breadcrumbs.shift()
        }

        breadcrumbs[0].name = '...'
      }

      return breadcrumbs
    }
  },
  methods: {
    ...mapMutations([ 'setHash', 'resetSelected', 'updateRequest', 'setLoading' ]),
    base64: function (name) {
      return window.btoa(unescape(encodeURIComponent(name)))
    },
    fetchData: async function () {
      // Reset view information.
      this.$store.commit('setReload', false)
      this.$store.commit('resetSelected')
      this.$store.commit('multiple', false)
      this.$store.commit('closeHovers')

      // Set loading to true and reset the error.
      this.setLoading(true)
      this.error = null

      try {
        let file = await api.getHash(encodeURIComponent(this.$route.params.pathMatch))
        this.path = file.path
        if (file.isDir) file.items = file.items.map((item, index) => {
          item.index = index
          item.url = `/share/${this.hash}${this.path}/${encodeURIComponent(item.name)}`
          return item
        })
        this.updateRequest(file)
        this.setLoading(false)
      } catch (e) {
        this.error = e
      }
    },
    keyEvent (event) {
      // Esc!
      if (event.keyCode === 27) {
        // If we're on a listing, unselect all
        // files and folders.
        if (this.selectedCount > 0) {
          this.resetSelected()
        }
      }
    },
    toggleMultipleSelection () {
      this.$store.commit('multiple', !this.multiple)
    }
  }
}
</script>
