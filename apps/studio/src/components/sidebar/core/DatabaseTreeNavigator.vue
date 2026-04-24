<template>
  <div class="database-tree-navigator">
    <div class="tree-section">
      <div class="section-header">
        <i class="material-icons">dns</i>
        <span class="expand">Connections</span>
        <a
          class="header-action"
          title="New folder"
          @click.prevent="openCreateFolderModal"
        >
          <i class="material-icons">create_new_folder</i>
        </a>
      </div>
      <nav class="tree-list">
        <Draggable
          :list="lonelyConnections"
          group="connections"
          ghost-class="drag-ghost"
          :animation="150"
          @start="onDragStart($event, lonelyConnections)"
          @end="draggingConnection = null"
          @change="onConnectionDrop($event, null, lonelyConnections)"
        >
          <connection-tree-row
            v-for="conn in lonelyConnections"
            :key="`lonely-${conn.id}-${conn.workspaceId}`"
            :connection="conn"
            :active="isActive(conn)"
            @select="selectConnection"
            @contextmenu.native.prevent="showConnectionContextMenu($event, conn)"
          />
        </Draggable>

        <sidebar-folder
          v-for="{ folder, connections: folderConnections } in folderGroups"
          :key="`folder-${folder.id}-${folderConnections.length}`"
          :title="folder.name"
          :expanded-initially="folderHasActive(folderConnections)"
          placeholder="No connections"
          @contextmenu.native.prevent.stop="showFolderContextMenu($event, folder)"
          @header-drop="onFolderHeaderDrop(folder)"
        >
          <Draggable
            :list="folderConnections"
            group="connections"
            ghost-class="drag-ghost"
            :animation="150"
            class="folder-draggable-area"
            @start="onDragStart($event, folderConnections)"
            @end="draggingConnection = null"
            @change="onConnectionDrop($event, folder, folderConnections)"
          >
            <connection-tree-row
              v-for="conn in folderConnections"
              :key="`f-${folder.id}-${conn.id}-${conn.workspaceId}`"
              :connection="conn"
              :active="isActive(conn)"
              @select="selectConnection"
              @contextmenu.native.prevent="showConnectionContextMenu($event, conn)"
            />
          </Draggable>
        </sidebar-folder>

        <div
          v-if="!lonelyConnections.length && !folderGroups.length"
          class="empty-msg"
        >
          No saved connections
        </div>
      </nav>
    </div>

    <div
      v-if="activeConfig"
      class="entities-section"
    >
      <div class="section-header">
        <i class="material-icons">table_view</i>
        <span class="truncate">{{ activeConfig.name || 'Tables' }}</span>
      </div>
      <surreal-namespace-dropdown
        v-if="connectionType === 'surrealdb'"
        @namespaceSelected="namespaceSelected"
      />
      <database-dropdown @databaseSelected="databaseSelected" />
      <div class="entities-content">
        <table-list />
      </div>
    </div>

    <portal to="modals">
      <modal
        class="vue-dialog beekeeper-modal"
        name="db-tree-folder-modal"
        @closed="folderModalName = ''; folderModalItem = null"
        @opened="$nextTick(() => $refs.folderNameInput && $refs.folderNameInput.focus())"
        height="auto"
        :scrollable="true"
      >
        <form @submit.prevent="submitFolderModal">
          <div class="dialog-content" v-kbd-trap="true">
            <div class="dialog-c-title">
              {{ folderModalItem ? 'Rename Folder' : 'New Folder' }}
            </div>
            <div class="form-group">
              <label>Folder Name</label>
              <input
                ref="folderNameInput"
                v-model="folderModalName"
                type="text"
                placeholder="Folder name"
                @keydown.esc.prevent="$modal.hide('db-tree-folder-modal')"
              >
            </div>
          </div>
          <div class="vue-dialog-buttons flex-between">
            <span class="left" />
            <span class="right">
              <button class="btn btn-flat" type="button" @click.prevent="$modal.hide('db-tree-folder-modal')">Cancel</button>
              <button class="btn btn-primary" type="submit" :disabled="!folderModalName.trim()">
                {{ folderModalItem ? 'Rename' : 'Create' }}
              </button>
            </span>
          </div>
        </form>
      </modal>
    </portal>
  </div>
</template>

<script>
import _ from 'lodash'
import Draggable from 'vuedraggable'
import { mapState, mapGetters } from 'vuex'
import { isUltimateType } from '@/common/interfaces/IConnection'
import SidebarFolder from '@/components/common/SidebarFolder.vue'
import ConnectionTreeRow from './ConnectionTreeRow.vue'
import TableList from './TableList.vue'
import DatabaseDropdown from './DatabaseDropdown.vue'
import SurrealNamespaceDropdown from './SurrealNamespaceDropdown.vue'
import rawLog from '@bksLogger'

const log = rawLog.scope('DatabaseTreeNavigator')

export default {
  name: 'DatabaseTreeNavigator',
  components: {
    SidebarFolder,
    ConnectionTreeRow,
    TableList,
    DatabaseDropdown,
    SurrealNamespaceDropdown,
    Draggable,
  },
  data() {
    return {
      folderModalName: '',
      folderModalItem: null,
      draggingConnection: null,
    }
  },
  computed: {
    ...mapState({
      activeConfig: state => state.usedConfig,
      connectionType: state => state.connectionType,
    }),
    ...mapState('data/connectionFolders', { folders: 'items' }),
    ...mapGetters({
      savedConnections: 'data/connections/filteredConnections',
      isUltimate: 'isUltimate',
    }),
    rootFolders() {
      return (this.folders || []).filter((f) => !f.parentId)
    },
    folderGroups() {
      return _.sortBy(this.rootFolders, (f) => (f.name || '').toLowerCase()).map((folder) => ({
        folder,
        connections: this.connectionsInFolder(folder.id),
      }))
    },
    lonelyConnections() {
      return _.sortBy(
        (this.savedConnections || []).filter((c) => !c.connectionFolderId),
        (c) => (c.name || '').toLowerCase()
      )
    },
  },
  methods: {
    connectionsInFolder(folderId) {
      return _.sortBy(
        (this.savedConnections || []).filter((c) => c.connectionFolderId === folderId),
        (c) => (c.name || '').toLowerCase()
      )
    },
    isActive(conn) {
      return (
        this.activeConfig &&
        conn.id === this.activeConfig.id &&
        conn.workspaceId === this.activeConfig.workspaceId
      )
    },
    folderHasActive(connections) {
      return connections.some((c) => this.isActive(c))
    },
    async selectConnection(config) {
      if (this.isActive(config)) return
      if (!this.isUltimate && isUltimateType(config?.connectionType)) {
        this.$noty.error('Cannot switch to Ultimate only connection.')
        return
      }
      this.$store.commit('connectionSwitching', true)
      try {
        if (this.activeConfig) {
          this.$store.commit('tabs/set', [])
          await this.$store.dispatch('disconnect')
        }
        const { auth, cancelled } = await this.$bks.unlock()
        if (cancelled) return
        await this.$store.dispatch('connect', { config, auth })
      } catch (ex) {
        log.error(ex)
        this.$noty.error('Error establishing a connection')
      } finally {
        this.$store.commit('connectionSwitching', false)
      }
    },
    async databaseSelected(db) {
      this.$store.dispatch('changeDatabase', db).catch((e) => {
        this.$noty.error(e.message)
      })
    },
    async namespaceSelected(ns) {
      this.$store.dispatch('changeNamespace', ns).catch((e) => {
        this.$noty.error(e.message)
      })
    },

    openCreateFolderModal() {
      this.folderModalItem = null
      this.folderModalName = ''
      this.$modal.show('db-tree-folder-modal')
    },
    showFolderContextMenu(event, folder) {
      this.$bks.openMenu({
        event,
        item: folder,
        options: [
          {
            name: 'Rename',
            handler: ({ item }) => {
              this.folderModalItem = item
              this.folderModalName = item.name
              this.$modal.show('db-tree-folder-modal')
            },
          },
          {
            name: 'Delete',
            handler: async ({ item }) => {
              try {
                await this.$store.dispatch('data/connectionFolders/remove', item)
                this.$noty.success(`Folder "${item.name}" deleted`)
              } catch (ex) {
                this.$noty.error(`Could not delete folder: ${ex.message}`)
              }
            },
          },
        ],
      })
    },
    showConnectionContextMenu(event, connection) {
      const moveToOptions = this.rootFolders
        .filter((f) => f.id !== connection.connectionFolderId)
        .map((folder) => ({
          name: `Move to ${folder.name}`,
          handler: () => this.moveConnectionToFolder(connection, folder),
        }))

      const options = []
      if (connection.connectionFolderId) {
        options.push({
          name: 'Move to top level',
          handler: () => this.moveConnectionToFolder(connection, null),
        })
      }
      options.push(...moveToOptions)
      if (!options.length) return

      this.$bks.openMenu({ event, item: connection, options })
    },
    async moveConnectionToFolder(connection, folder) {
      try {
        await this.$store.dispatch('data/connections/reorder', {
          item: connection,
          connectionFolderId: folder?.id ?? null,
          position: { before: null },
        })
      } catch (ex) {
        this.$noty.error(`Move error: ${ex.userMessage ?? ex.message}`)
      }
    },
    onDragStart(event, list) {
      this.draggingConnection = list[event.oldIndex]
    },
    cloudRelativePosition(list, newIndex) {
      const prev = list[newIndex - 1]
      const next = list[newIndex + 1]
      if (prev) return { after: prev.id }
      if (next) return { before: next.id }
      return { before: null }
    },
    async onConnectionDrop(event, folder, currentList) {
      try {
        if (event.added) {
          const { element: item, newIndex } = event.added
          await this.$store.dispatch('data/connections/reorder', {
            item,
            connectionFolderId: folder?.id ?? null,
            position: this.cloudRelativePosition(currentList, newIndex),
          })
        } else if (event.moved) {
          const { element: item, newIndex } = event.moved
          await this.$store.dispatch('data/connections/reorder', {
            item,
            position: this.cloudRelativePosition(currentList, newIndex),
          })
        }
      } catch (ex) {
        this.$noty.error(`Move error: ${ex.userMessage ?? ex.message}`)
      }
    },
    async onFolderHeaderDrop(folder) {
      if (!this.draggingConnection) return
      try {
        await this.$store.dispatch('data/connections/reorder', {
          item: this.draggingConnection,
          connectionFolderId: folder.id,
          position: { before: null },
        })
      } catch (ex) {
        this.$noty.error(`Move error: ${ex.userMessage ?? ex.message}`)
      }
    },
    async submitFolderModal() {
      const name = this.folderModalName.trim()
      if (!name) return
      try {
        if (this.folderModalItem) {
          await this.$store.dispatch('data/connectionFolders/save', { ...this.folderModalItem, name })
        } else {
          await this.$store.dispatch('data/connectionFolders/save', { id: null, name, parentId: null })
        }
        this.$modal.hide('db-tree-folder-modal')
      } catch (ex) {
        this.$noty.error(`Could not save folder: ${ex.message}`)
      }
    },
  },
}
</script>

<style lang="scss" scoped>
.database-tree-navigator {
  display: flex;
  flex-direction: column;
  height: 100%;
  width: 100%;
  overflow: hidden;
}

.tree-section {
  display: flex;
  flex-direction: column;
  flex-shrink: 0;
  max-height: 35%;
  border-bottom: 1px solid rgba(127, 127, 127, 0.18);
}

.tree-list {
  overflow-y: auto;
  padding: 0.25rem 0;
}

.entities-section {
  display: flex;
  flex-direction: column;
  flex: 1;
  min-height: 0;
  overflow: hidden;
}

.entities-content {
  flex: 1;
  min-height: 0;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  padding-left: 0.5rem;
}

.section-header {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  padding: 0.4rem 0.6rem;
  font-size: 0.7rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  opacity: 0.65;
  font-weight: 600;
  flex-shrink: 0;

  .material-icons {
    font-size: 14px;
  }
}

.header-action {
  cursor: pointer;
  opacity: 0.6;
  display: inline-flex;
  align-items: center;
  padding: 2px 4px;
  border-radius: 3px;

  &:hover {
    opacity: 1;
    background: rgba(127, 127, 127, 0.18);
  }

  .material-icons {
    font-size: 16px;
  }
}

.empty-msg {
  padding: 0.75rem;
  text-align: center;
  font-size: 0.8rem;
  opacity: 0.5;
}

::v-deep .drag-ghost {
  opacity: 0.4;
}

.folder-draggable-area {
  min-height: 12px;
  padding-left: 1.25rem;
}
</style>
