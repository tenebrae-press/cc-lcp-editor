<template>
  <v-container>
    <v-row justify="space-around" align="center" class="mb-3">
      <v-col cols="5">
        <v-file-input
          v-model="lcpFile"
          filled
          color="primary"
          label="Load LCP"
          accept=".lcp,.zip"
          show-size
          hide-details
          :loading="loading"
          @click:clear="clearLcp()"
          @change="loadLcp($event)" />
      </v-col>
      <v-col cols="5">
        <v-btn large block color="primary" @click="createNew">Create New LCP</v-btn>
      </v-col>
    </v-row>

    <v-card v-if="!loaded" :key="loaded" color="grey darken-2">
      <v-card-text class="text--disabled text-center">No LCP loaded</v-card-text>
    </v-card>

    <v-card v-else>
      <v-toolbar dense color="pink darken-3" dark>
        <span class="text-h4">{{ lcp.lcp_manifest.name }}</span>
      </v-toolbar>
      <v-card-text>
        <v-alert outlined text color="primary">
          <div class="text-h5">LCP Manifest</div>
          <v-row>
            <v-col>
              <v-row dense justify="space-around">
                <v-col cols="3">
                  <v-text-field v-model="lcp.lcp_manifest.name" label="Name" />
                </v-col>
                <v-col cols="3">
                  <v-text-field v-model="lcp.lcp_manifest.author" label="Author" />
                </v-col>
                <v-col cols="2">
                  <v-text-field v-model="lcp.lcp_manifest.item_prefix" label="Item ID Prefix" />
                </v-col>
                <v-col cols="2">
                  <v-text-field v-model="lcp.lcp_manifest.version" label="LCP Version" />
                </v-col>
                <v-col cols="5">
                  <v-text-field v-model="lcp.lcp_manifest.website" label="Website URL" />
                </v-col>
                <v-col cols="5">
                  <v-text-field v-model="lcp.lcp_manifest.image_url" label="Preview Image URL" />
                </v-col>
                <v-col cols="12">
                  <v-textarea
                    v-model="lcp.lcp_manifest.description"
                    outlined
                    auto-grow
                    rows="3"
                    label="Description" />
                </v-col>
              </v-row>
            </v-col>
            <v-col v-show="!!lcp.lcp_manifest.image_url" cols="auto">
              Image Preview
              <v-img :src="lcp.lcp_manifest.image_url" max-width="300" contain />
            </v-col>
          </v-row>
        </v-alert>
        <div class="text-h6 mb-2">LCP Contents:</div>
        <v-row justify="space-around">
          <v-col cols="10">
            <v-btn x-large block color="primary darken-3" to="editor/manufacturers">
              Manufacturers & Licenses
            </v-btn>
            <br />
            <div class="text-center mt-n4">
              {{ manuCount }} Manufacturers ({{ catLength('manufacturers') }} new) with
              {{ catLength('frames') }} Licenses, containing {{ catLength('weapons') }} Weapons,
              {{ catLength('systems') }} Systems and {{ catLength('mods') }} Weapon Mods
            </div>
          </v-col>
        </v-row>

        <v-row justify="space-around">
          <v-col v-for="(t, i) in categories" :key="`player_btn_${i}`" cols="2">
            <v-btn large block color="primary darken-3" :to="`editor/${t}`">
              {{ t.replace('_', ' ') }}
              <span class="item-count">({{ catLength(t) }})</span>
            </v-btn>
          </v-col>
          <v-col cols="2">
            <v-btn :to="'editor/tables'" large block color="primary darken-3">
              {{ 'Tables' }}
            </v-btn>
          </v-col>
        </v-row>
        <v-divider class="my-4" />
        <v-alert outlined color="red" icon="mdi-alert">
          The editors below are only compatible with the forthcoming GM tools update. They
          <b>SHOULD NOT</b>
          be used to create NPC data for the current version of COMP/CON
        </v-alert>
        <v-row justify="space-around">
          <v-col v-for="(t, i) in gmCategories" :key="`gm_btn_${i}`" cols="2">
            <v-btn large block color="primary darken-3" :to="`editor/${t}`">
              {{ t.replace('_', ' ') }}
              <span class="item-count">({{ catLength(t) }})</span>
            </v-btn>
          </v-col>
        </v-row>
      </v-card-text>
      <v-divider class="my-2" />
      <v-card-actions>
        <v-btn x-large block color="success" @click="exportLCP">Export LCP</v-btn>
      </v-card-actions>
    </v-card>
  </v-container>
</template>

<script lang="ts">
import Vue from 'vue'
import _ from 'lodash'
import PromisifyFileReader from 'promisify-file-reader'
import JSZip from 'jszip'
import { saveAs } from 'file-saver'
import Lancer, {
  ILCPContent,
  IManifest,
  LCPContentKeys,
  LCPContentTypes,
} from '@tenebrae-press/lancer-types'

const skipCategories = [
  'manufacturers',
  'frames',
  'weapons',
  'systems',
  'mods',
  'core_bonuses',
  'npc_features',
]
const gmCategories = ['npc_classes', 'npc_templates']

export default Vue.extend({
  name: 'Home',
  data: () => ({
    lcpFile: null,
    loading: false,
    categories: Lancer.LCP_CONTENT_KEYS.filter(
      key => !gmCategories.includes(key) && !skipCategories.includes(key)
    ),
    gmCategories: Lancer.LCP_CONTENT_KEYS.filter(
      key => gmCategories.includes(key) && !skipCategories.includes(key)
    ),
  }),
  computed: {
    loaded(): boolean {
      return this.$store.getters.loaded
    },
    lcp(): ILCPContent {
      return this.$store.getters.lcp
    },
    manuCount(): number {
      if (!this.lcp.manufacturers && !this.lcp.frames) return 0
      const m =
        this.lcp.manufacturers && this.lcp.manufacturers.length
          ? this.lcp.manufacturers.map(x => x.id)
          : []
      const f = this.lcp.frames ? _.uniq(this.lcp.frames.map(x => x.source)) : []
      return _.uniq(m.concat(f)).length
    },
  },
  methods: {
    catLength(type: LCPContentKeys) {
      if (this.lcp[type]) return this.lcp[type]?.length
      return 0
    },

    async loadLcp(file: HTMLInputElement) {
      this.loading = true

      if (!file) return

      const fileData = await PromisifyFileReader.readAsBinaryString(file)
      try {
        this.$store.dispatch('loadLcp', fileData).then(() => {
          this.loading = false
        })
      } catch (e: unknown) {
        console.error(e)
        this.loading = false
      }
    },
    clearLcp() {
      this.$store.dispatch('clearLcp')
    },
    createNew() {
      this.$store.dispatch('clearLcp')
      this.$store.dispatch('newLcp')
    },
    exportLCP() {
      const filename = `${this.lcp.lcp_manifest.name.toLowerCase().replaceAll(' ', '-')}_${
        this.lcp.lcp_manifest.version
      }.lcp`
      const zip = new JSZip()
      Lancer.LCP_CONTENT_KEYS.forEach((key: LCPContentKeys) => {
        if (this.lcp[key] && this.lcp[key]?.length) {
          zip.file(`${key}.json`, this.prepareJSON(this.lcp[key] ?? []))
        }
      })
      zip.file('lcp_manifest.json', this.prepareJSON(this.lcp.lcp_manifest))
      if (this.lcp.tables) {
        zip.file('tables.json', this.prepareJSON(this.lcp.tables))
      }
      zip.generateAsync({ type: 'blob' }).then(function (blob) {
        saveAs(blob, filename)
      })
    },
    prepareJSON(obj: LCPContentTypes | IManifest | Record<string, unknown>): string {
      const d = JSON.stringify(obj)
      // tiptap's default <p> wrapping doesn't look good in C/C
      d.replaceAll('<p', '<div')
      d.replaceAll('</p', '</div')

      return d
    },
  },
})
</script>

<style scoped>
.item-count {
  font-size: 12px;
  font-weight: 100;
  margin-left: 4px;
}
</style>
