<template>
  <div>
    <b-modal :active.sync="active" :width="1200">
      <section class="dialog section">
        <loading-tab v-if="loading"></loading-tab>
        <template v-else>
          <b-message type="is-danger" v-if="error">{{ error }}</b-message>
          <article class="content" v-if="content">
            <plist v-if="type == 'plist'" title="Plist Reader" :content="content" :rootName="file.name"></plist>
            <hex-view v-if="type == 'text'" :raw="content"></hex-view>
            <database v-if="type == 'sql'" :file="file" :content="content"></database>
            <p v-if="type == 'image'" class="image"><img :src="content"></p>
          </article>
        </template>
      </section>
    </b-modal>

  </div>
</template>

<script>
import { mapGetters } from 'vuex'
import { GET_SOCKET } from '~/vuex/types'
import { download } from '~/lib/utils'
import LoadingTab from '~/components/LoadingTab.vue'
import Plist from '~/components/Plist.vue'
import HexView from '~/components/HexView.vue'
import Database from '~/components/Database.vue'


export default {
  components: { LoadingTab, Plist, HexView, Database },
  props: {
    type: String,
    open: Boolean,
    file: Object,
  },
  computed: {
    active: {
      set(val) {
        this.$emit('update:open', val)
      },
      get() {
        return this.open
      }
    },
    ...mapGetters({
      socket: GET_SOCKET,
    })
  },
  data() {
    return {
      content: null,
      loading: false,
      error: null,
    }
  },
  watch: {
    open(val, old) {
      if (!old && val && this.file)
        this.view(this.file.path)
    },
    file(val, old) {
      if (val && this.open) {
        this.view(val.path)
      }
    }
  },
  mounted() {
    if (this.file && this.file.path)
      this.view(this.file.path)
  },
  beforeDestroy() {
    if (this.type === 'image' && this.content)
      URL.revokeObjectURL(this.content)
  },
  methods: {
    view(path) {
      this.error = ''
      if (this.type === 'plist' || this.type === 'text') {
        this.loading = true
        this.socket.call(this.type, path)
          .then(content => this.content = content)
          .catch(err => this.error = err)
          .finally(() => this.loading = false)
      } else if (this.type === 'sql') {
        // hack: force component to load
        this.$nextTick(() => this.content = this.file.path)
      } else if (this.type === 'image') {
        this.loading = true
        download(this.socket, this.file, 'image/*')
          .then(url => this.content = url)
          .finally(() => this.loading = false)
      }
    }
  }
}
</script>
