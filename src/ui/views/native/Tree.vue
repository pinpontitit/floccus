<template>
  <div
    v-touch="{down: () => onSwipeDown()}"
    style="height:100%">
    <Drawer :visible.sync="drawer" />
    <v-app-bar
      absolute
      app>
      <v-app-bar-nav-icon
        v-if="!tree || currentFolderId === tree.id"
        class="mr-2 ml-n2"
        @click="drawer = !drawer" />
      <v-btn
        v-else
        icon
        class="mr-2 ml-n2"
        @click="goBack">
        <v-icon>mdi-arrow-left</v-icon>
      </v-btn>
      <v-text-field
        :value="searchQuery"
        :label="!tree || currentFolderId === tree.id? t('LabelSearch') : t('LabelSearchfolder', [currentFolder.title])"
        solo
        flat
        dense
        clearable
        hide-details
        @input="onSearch" />
      <v-spacer />
      <v-btn
        icon
        :disabled="!currentAccount"
        :color="syncing? 'primary' : ''"
        @click="onTriggerSync">
        <v-icon
          :class="{'sync--active': Boolean(syncing)}">
          mdi-sync
        </v-icon>
      </v-btn>
      <v-menu
        bottom
        left>
        <template #activator="{ on, attrs }">
          <v-btn
            icon
            v-bind="attrs"
            v-on="on">
            <v-icon>{{ sortIcons[sortBy] }}</v-icon>
          </v-btn>
        </template>

        <v-list>
          <v-list-item @click="sortBy = 'title'">
            <v-list-item-avatar>
              <v-icon>{{ sortIcons['title'] }}</v-icon>
            </v-list-item-avatar>
            <v-list-item-title>
              {{ t('LabelSorttitle') }}
            </v-list-item-title>
          </v-list-item>
          <v-list-item @click="sortBy = 'url'">
            <v-list-item-avatar>
              <v-icon>{{ sortIcons['url'] }}</v-icon>
            </v-list-item-avatar>
            <v-list-item-title>
              {{ t('LabelSorturl') }}
            </v-list-item-title>
          </v-list-item>
          <v-list-item @click="sortBy = 'index'">
            <v-list-item-avatar>
              <v-icon>{{ sortIcons['index'] }}</v-icon>
            </v-list-item-avatar>
            <v-list-item-title>
              {{ t('LabelSortcustom') }}
            </v-list-item-title>
          </v-list-item>
        </v-list>
      </v-menu>
      <v-btn
        v-if="currentAccount"
        icon
        :to="{name: routes.ACCOUNT_OPTIONS, params:{accountId: currentAccount? currentAccount.id : 0}}">
        <v-icon>mdi-cog</v-icon>
      </v-btn>
    </v-app-bar>
    <v-main>
      <v-alert
        v-if="Boolean(syncError)"
        dense
        outlined
        text
        type="warning"
        class="ma-1">
        {{ syncError }}
      </v-alert>
      <v-progress-linear
        v-if="syncProgress"
        :value="syncProgress * 100 || 0"
        color="blue darken-1" />
      <v-progress-circular
        v-if="loading"
        indeterminate
        color="blue darken-1"
        class="loading" />
      <v-list
        v-else-if="currentFolder && items && items.length"
        two-line
        class="mb-10">
        <template v-for="item in items">
          <v-list-item
            :key="item.type+item.id"
            class="pl-3"
            dense
            @click="clickItem(item)">
            <v-list-item-avatar>
              <v-icon
                v-if="item.type === 'folder'"
                color="blue darken-1"
                large>
                mdi-folder
              </v-icon>
              <FaviconImage
                v-else
                :url="item.url"
                :use-network="useNetwork" />
            </v-list-item-avatar>

            <v-list-item-content>
              <v-list-item-title>{{ item.title }}</v-list-item-title>
              <v-list-item-subtitle v-if="item.type === 'bookmark'">
                {{ item.url | hostname }}
              </v-list-item-subtitle>
            </v-list-item-content>

            <v-list-item-action>
              <v-menu
                bottom
                left>
                <template #activator="{ on, attrs }">
                  <v-btn
                    icon
                    v-bind="attrs"
                    v-on="on">
                    <v-icon>mdi-dots-vertical</v-icon>
                  </v-btn>
                </template>

                <v-list>
                  <v-list-item @click="editItem(item)">
                    <v-list-item-avatar>
                      <v-icon>mdi-pencil</v-icon>
                    </v-list-item-avatar>
                    <v-list-item-title>
                      {{ t('LabelEdititem') }}
                    </v-list-item-title>
                  </v-list-item>
                  <v-list-item @click="deleteItem(item)">
                    <v-list-item-avatar>
                      <v-icon>mdi-delete</v-icon>
                    </v-list-item-avatar>
                    <v-list-item-title>
                      {{ t('LabelDeleteitem') }}
                    </v-list-item-title>
                  </v-list-item>
                </v-list>
              </v-menu>
            </v-list-item-action>
          </v-list-item>
          <v-divider
            :key="String(item.id)+item.type+'divider'" />
        </template>
      </v-list>
      <v-card
        v-else
        flat
        tile
        :style="{margin: '10vh auto', width: '90vw'}">
        <img
          src="icons/tree-swing.svg"
          :style="{width: '95%', maxHeight: '40vh'}">
        <h3 class="text-center headline mt-5">
          {{ t('LabelNobookmarks') }}
        </h3>
      </v-card>
      <v-speed-dial
        v-model="fab"
        fixed
        bottom
        right>
        <template #activator>
          <v-btn
            v-model="fab"
            color="blue darken-1"
            dark
            fab>
            <v-icon v-if="fab">
              mdi-close
            </v-icon>
            <v-icon v-else>
              mdi-plus
            </v-icon>
          </v-btn>
        </template>
        <v-btn
          color="blue darken-1"
          dark
          small
          fab
          @click="addFolder">
          <v-icon>mdi-folder</v-icon>
        </v-btn>
        <v-btn
          color="blue darken-1"
          dark
          small
          fab
          @click="addBookmark">
          <v-icon>mdi-star</v-icon>
        </v-btn>
      </v-speed-dial>
    </v-main>

    <DialogEditBookmark
      v-if="isAddingBookmark"
      :is-new="true"
      :display.sync="isAddingBookmark"
      :tree="tree"
      :parent-folder="currentFolderId"
      @save="createBookmark($event)" />
    <DialogEditFolder
      v-if="isAddingFolder"
      :is-new="true"
      :display.sync="isAddingFolder"
      :tree="tree"
      :parent-folder="currentFolderId"
      @save="createFolder($event)" />
    <DialogEditBookmark
      v-if="isEditingBookmark"
      :is-new="false"
      :bookmark="currentlyEditedBookmark"
      :tree="tree"
      :display.sync="isEditingBookmark"
      @save="editBookmark($event)" />
    <DialogEditFolder
      v-if="isEditingFolder"
      :is-new="false"
      :folder="currentlyEditedFolder"
      :display.sync="isEditingFolder"
      :tree="tree"
      @save="editFolder($event)" />
  </div>
</template>

<script>
import Drawer from '../../components/native/Drawer'
import DialogEditFolder from '../../components/native/DialogEditFolder'
import DialogEditBookmark from '../../components/native/DialogEditBookmark'
import FaviconImage from '../../components/native/FaviconImage'
import { routes } from '../../NativeRouter'
import { Bookmark, Folder } from '../../../lib/Tree'
import { actions } from '../../store/definitions'
import { App } from '@capacitor/app'
import sortBy from 'lodash/sortBy'

export default {
  name: 'Tree',
  components: { FaviconImage, DialogEditBookmark, DialogEditFolder, Drawer },
  filters: {
    hostname(url) {
      return new URL(url).hostname
    }
  },
  data() {
    return {
      currentFolderId: 0,
      drawer: false,
      searchQuery: '',
      isEditingFolder: false,
      currentlyEditedFolder: null,
      isEditingBookmark: false,
      currentlyEditedBookmark: null,
      isAddingBookmark: false,
      isAddingFolder: false,
      fab: false,
      searchDebounceTimer: null,
      sortIcons: {
        title: 'mdi-sort-alphabetical-ascending',
        url: 'mdi-sort-bool-ascending',
        index: 'mdi-sort-ascending'
      },
      sortBy: 'index',
      syncProgress: 0,
    }
  },
  computed: {
    id() {
      return this.$route.params.accountId
    },
    loading() {
      return (!this.$store.state.accounts[this.id] || !this.$store.state.accounts[this.id].data || !Object.keys(this.$store.state.accounts[this.id].data).length || !this.tree)
    },
    tree() {
      return this.$store.state.tree
    },
    syncing() {
      if (this.loading) {
        return false
      }
      return this.$store.state.accounts[this.id].data.syncing
    },
    syncError() {
      if (this.loading) {
        return false
      }
      return this.$store.state.accounts[this.id].data.error
    },
    useNetwork() {
      if (this.loading) {
        return false
      }
      return this.$store.state.accounts[this.id].data.allowNetwork
    },
    items() {
      if (!this.currentFolder) {
        return []
      }
      let items
      if (this.searchQuery && this.searchQuery.length >= 2) {
        items = this.search(this.searchQuery.toLowerCase().trim(), this.currentFolder)
      } else {
        items = this.currentFolder.children
      }
      if (this.sortBy !== 'index') {
        return sortBy(items, [(item) => {
          if (this.sortBy === 'url' && item.url) {
            return new URL(item[this.sortBy]).hostname
          }
          return item[this.sortBy].toLowerCase()
        }])
      } else {
        return items
      }
    },
    routes() {
      return routes
    },
    currentAccount() {
      return this.$store.state.accounts[this.id]
    },
    currentFolder() {
      return this.findItem(this.currentFolderId, this.tree)
    },
  },
  watch: {
    async $route() {
      await this.$store.dispatch(actions.LOAD_TREE, this.$route.params.accountId)
    },
    async syncing(current, previous) {
      if (!current && previous) {
        this.syncProgress = 1
        setTimeout(() => { this.syncProgress = 0 }, 1000)
      } else {
        this.syncProgress = current
      }
      if (!current) {
        await this.$store.dispatch(actions.LOAD_TREE, this.$route.params.accountId)
      }
    },
  },
  mounted() {
    this.$store.dispatch(actions.LOAD_TREE, this.$route.params.accountId)
  },
  backButton() {
    this.goBack()
  },
  methods: {
    clickItem(item) {
      if (item.url) {
        window.location = item.url
      } else {
        this.searchQuery = ''
        this.currentFolderId = item.id
      }
    },
    findItem(id, tree) {
      if (!tree) {
        return null
      }
      if (tree.id === id) {
        return tree
      }
      if (tree.url) {
        return false
      }
      return tree.findFolder(id)
    },
    onSearch(query) {
      clearTimeout(this.searchDebounceTimer)
      this.searchDebounceTimer = setTimeout(() => {
        this.searchQuery = query
      }, 500)
    },
    search(query, tree) {
      return Object.values(tree.index.bookmark).filter(item => {
        const matchTitle = item.title ? query.split(' ').every(term => item.title.toLowerCase().includes(term)) : false
        const matchUrl = query.split(' ').every(term => item.url.toLowerCase().includes(term))
        return matchUrl || matchTitle
      })
    },
    goBack() {
      if (this.isAddingBookmark) {
        this.isAddingBookmark = false
        return
      }
      if (this.isEditingBookmark) {
        this.isEditingBookmark = false
        return
      }
      if (this.isAddingFolder) {
        this.isAddingFolder = false
        return
      }
      if (this.isEditingFolder) {
        this.isEditingFolder = false
        return
      }
      if (this.searchQuery) {
        this.searchQuery = ''
        return
      }
      if (typeof this.currentFolder.parentId !== 'undefined') {
        this.currentFolderId = this.currentFolder.parentId
        return
      }
      App.exitApp()
    },
    editItem(item) {
      if (item.url) {
        this.currentlyEditedBookmark = item
        this.isEditingBookmark = true
      } else {
        this.currentlyEditedFolder = item
        this.isEditingFolder = true
      }
    },
    deleteItem(item) {
      if (item.type === 'bookmark') {
        this.$store.dispatch(actions.DELETE_BOOKMARK, {
          accountId: this.id,
          bookmark: item
        })
      } else {
        this.$store.dispatch(actions.DELETE_FOLDER, {
          accountId: this.id,
          folder: item
        })
      }
    },
    addBookmark() {
      this.isAddingBookmark = true
    },
    createBookmark(props) {
      this.$store.dispatch(actions.CREATE_BOOKMARK, {
        accountId: this.id,
        bookmark: new Bookmark({ id: null, parentId: this.currentFolderId, ...props })
      })
    },
    addFolder() {
      this.isAddingFolder = true
    },
    createFolder(props) {
      this.$store.dispatch(actions.CREATE_FOLDER, {
        accountId: this.id,
        folder: new Folder({...props, id: null, parentId: this.currentFolderId})
      })
    },
    editFolder(props) {
      this.$store.dispatch(actions.EDIT_FOLDER, {
        accountId: this.id,
        folder: new Folder({...this.currentlyEditedFolder, ...props})
      })
    },
    editBookmark(props) {
      this.$store.dispatch(actions.EDIT_BOOKMARK, {
        accountId: this.id,
        bookmark: new Bookmark({...this.currentlyEditedBookmark, ...props})
      })
    },
    onTriggerSync() {
      if (this.syncing) {
        return
      }
      this.currentAccount.data.syncing = 0.0001 // faaast
      this.$store.dispatch(actions.TRIGGER_SYNC, this.id)
    },
    onSwipeDown() {
      if (window.scrollY === 0) {
        this.onTriggerSync()
      }
    }
  }
}
</script>

<style scoped>
.loading {
  margin: 45vh 45vw;
}

.sync--active {
  animation: spin 2s infinite linear;
}

@keyframes spin {
  0% {
    transform: rotate(360deg);
  }
  99.9% {
    transform: rotate(0deg);
  }
}
</style>
