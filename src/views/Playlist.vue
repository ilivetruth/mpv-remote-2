<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-buttons slot="start">
          <ion-menu-button color="#fff"></ion-menu-button>
        </ion-buttons>
        <ion-title> Playlists </ion-title>
      </ion-toolbar>
      <ion-toolbar>
        <ion-segment v-model="selectedTab" @ionChange="onTabChange">
          <ion-segment-button value="current">
            <ion-label>Current Queue</ion-label>
          </ion-segment-button>
          <ion-segment-button value="saved">
            <ion-label>Saved</ion-label>
          </ion-segment-button>
        </ion-segment>
      </ion-toolbar>
    </ion-header>
    <ion-content :fullscreen="true">
      <!-- Current Queue Tab -->
      <div v-if="selectedTab === 'current'">
        <div style="display: flex; gap: 10px; margin: 10px;">
          <ion-button
            @click="onClearPlaylistClicked"
            size="small"
            :disabled="playerData.playlist.length == 0"
          >
            <ion-icon :icon="trashBin"></ion-icon>
            Clear
          </ion-button>
          <ion-button
            @click="onSaveCurrentPlaylistClicked"
            size="small"
            color="success"
            :disabled="playerData.playlist.length == 0"
          >
            <ion-icon :icon="save"></ion-icon>
            Save Queue
          </ion-button>
        </div>
        <ion-reorder-group
          @ionItemReorder="doReorder($event)"
          :disabled="playerData.playlist.length <= 1"
        >
          <ion-item
            lines="full"
            @click="onItemClicked(item)"
            v-for="item in playerData.playlist"
            :key="item.id"
          >
            <ion-icon
              class="playlistItemIndicator"
              v-if="item.current"
              slot="start"
              :icon="play"
            ></ion-icon>
            <ion-label class="ion-text-wrap">
              <p>
                {{
                  item.current
                    ? playerData["media-title"] || item.filename
                    : item.filename
                }}
              </p>
            </ion-label>
            <ion-button
              @click="onRemoveItemClicked(item)"
              fill="clear"
              slot="end"
            >
              <ion-icon
                style="color: white"
                slot="icon-only"
                :icon="trashBin"
              ></ion-icon>
            </ion-button>
            <ion-reorder slot="end"></ion-reorder>
          </ion-item>
        </ion-reorder-group>
      </div>

      <!-- Saved Playlists Tab -->
      <div v-else-if="selectedTab === 'saved'">
        <ion-list v-if="!viewingPlaylist">
          <ion-item
            lines="full"
            v-for="playlist in savedPlaylists"
            :key="playlist.id"
            @click="onSavedPlaylistClicked(playlist)"
          >
            <ion-label>
              <h2>{{ playlist.name }}</h2>
              <p>{{ new Date(playlist.created_date).toLocaleDateString() }}</p>
            </ion-label>
            <ion-button
              @click.stop="onLoadPlaylistClicked(playlist)"
              fill="clear"
              slot="end"
            >
              <ion-icon :icon="play" style="color: green;"></ion-icon>
            </ion-button>
            <ion-button
              @click.stop="onDeleteSavedPlaylistClicked(playlist)"
              fill="clear"
              slot="end"
            >
              <ion-icon :icon="trashBin" style="color: white;"></ion-icon>
            </ion-button>
          </ion-item>
          <ion-item v-if="savedPlaylists.length === 0" lines="none">
            <ion-label class="ion-text-center">
              <p>No saved playlists yet</p>
            </ion-label>
          </ion-item>
        </ion-list>

        <!-- Viewing a specific saved playlist -->
        <div v-else>
          <ion-toolbar>
            <ion-buttons slot="start">
              <ion-button @click="viewingPlaylist = null">
                <ion-icon :icon="arrowBack"></ion-icon>
                Back
              </ion-button>
            </ion-buttons>
            <ion-title size="small">{{ viewingPlaylist.name }}</ion-title>
            <ion-buttons slot="end">
              <ion-button @click="onLoadPlaylistClicked(viewingPlaylist)" color="success">
                <ion-icon :icon="play"></ion-icon>
                Load
              </ion-button>
            </ion-buttons>
          </ion-toolbar>
          <ion-reorder-group
            @ionItemReorder="doSavedPlaylistReorder($event)"
            :disabled="viewingPlaylist.entries.length <= 1"
          >
            <ion-item
              lines="full"
              v-for="entry in viewingPlaylist.entries"
              :key="entry.id"
            >
              <ion-label class="ion-text-wrap">
                <p>{{ entry.file_path }}</p>
              </ion-label>
              <ion-button
                @click="onRemoveSavedEntryClicked(entry)"
                fill="clear"
                slot="end"
              >
                <ion-icon :icon="trashBin" style="color: white;"></ion-icon>
              </ion-button>
              <ion-reorder slot="end"></ion-reorder>
            </ion-item>
          </ion-reorder-group>
        </div>
      </div>
    </ion-content>
    <playerController
      v-if="serverConfigured && isPlayerActive && connectedState"
    ></playerController>
  </ion-page>
</template>

<script>
import {
  IonPage,
  IonHeader,
  IonToolbar,
  IonButtons,
  IonMenuButton,
  IonTitle,
  IonContent,
  IonReorder,
  IonReorderGroup,
  IonItem,
  IonLabel,
  IonIcon,
  IonButton,
  IonSegment,
  IonSegmentButton,
  IonList,
  alertController,
} from "@ionic/vue";

import { play, add, remove, trashBin, save, arrowBack } from "ionicons/icons";
import { computed, ref, onMounted } from "vue";
import { useStore } from "vuex";
import playerController from "../components/playerController.vue";
import { apiInstance } from "../api";
export default {
  setup() {
    const store = useStore();
    const connectedState = computed(() => store.state.simpleapi.connected);
    const serverConfigured = computed(
      () => store.state.settings.settings.configured
    );
    const playerData = computed(() => store.state.simpleapi.playerData);
    const isPlayerActive = computed(() => {
      return playerData.value.filename ? true : false;
    });
    const DOUBLE_CLICK_THRESHOLD = 500;
    let lastOnStart = 0;

    // Tab management
    const selectedTab = ref('current');
    const savedPlaylists = ref([]);
    const viewingPlaylist = ref(null);

    const loadSavedPlaylists = async () => {
      try {
        const response = await apiInstance.get('/saved-playlists');
        savedPlaylists.value = response.data;
      } catch (error) {
        console.error('Failed to load saved playlists:', error);
      }
    };

    const onTabChange = () => {
      if (selectedTab.value === 'saved') {
        loadSavedPlaylists();
      }
      viewingPlaylist.value = null;
    };

    onMounted(() => {
      loadSavedPlaylists();
    });

    const doReorder = async (event) => {
      let fromIndex = event.detail.from;
      let toIndex = event.detail.to;
      // We moving element down.
      if (toIndex > fromIndex) {
        toIndex += 2;
      }
      apiInstance
        .post("playlist/move", null, { params: { fromIndex, toIndex } })
        .then(() => {
          event.detail.complete(true);
        })
        .catch(() => event.detail.complete(false));
    };

    const onClearPlaylistClicked = () => {
      apiInstance.post("/playlist/clear");
    };

    const onRemoveItemClicked = (item) => {
      apiInstance.post(`/playlist/remove/${item.index}`);
    };

    const onItemClicked = (item) => {
      /* TODO: Create double tap gesture somehow for ion-item.
        https://ionicframework.com/docs/utilities/gestures
        I'm using my own shitty solution for this, because dunno how to add all ion-item elements to gesture.
        But it works :-)*/
      const now = Date.now();

      if (Math.abs(now - lastOnStart) <= DOUBLE_CLICK_THRESHOLD) {
        const index = playerData.value.playlist.findIndex(
          (el) => el.id == item.id
        );

        if (index > -1) {
          apiInstance.post(`/playlist/play/${index}`);
          console.log(playerData.value.playlist);
        }
        lastOnStart = 0;
      } else {
        lastOnStart = now;
      }
    };

    const onSaveCurrentPlaylistClicked = async () => {
      const alert = await alertController.create({
        header: 'Save Playlist',
        message: 'Enter a name for this playlist',
        inputs: [
          {
            name: 'name',
            type: 'text',
            placeholder: 'Playlist name'
          }
        ],
        buttons: [
          {
            text: 'Cancel',
            role: 'cancel'
          },
          {
            text: 'Save',
            handler: async (data) => {
              if (data.name) {
                try {
                  const entries = playerData.value.playlist.map(item => item.filename);
                  await apiInstance.post('/saved-playlists', {
                    name: data.name,
                    entries: entries
                  });
                  store.dispatch('app/showToast', {
                    message: 'Playlist saved successfully',
                    duration: 2000
                  });
                  loadSavedPlaylists();
                } catch (error) {
                  console.error('Failed to save playlist:', error);
                  store.dispatch('app/showToast', {
                    message: 'Failed to save playlist',
                    duration: 2000
                  });
                }
              }
            }
          }
        ]
      });
      await alert.present();
    };

    const onSavedPlaylistClicked = async (playlist) => {
      try {
        const response = await apiInstance.get(`/saved-playlists/${playlist.id}`);
        viewingPlaylist.value = response.data;
      } catch (error) {
        console.error('Failed to load playlist details:', error);
      }
    };

    const onLoadPlaylistClicked = async (playlist) => {
      try {
        await apiInstance.post(`/saved-playlists/${playlist.id}/load`);
        store.dispatch('app/showToast', {
          message: `Loaded "${playlist.name}"`,
          duration: 2000
        });
        selectedTab.value = 'current';
      } catch (error) {
        console.error('Failed to load playlist:', error);
        store.dispatch('app/showToast', {
          message: 'Failed to load playlist',
          duration: 2000
        });
      }
    };

    const onDeleteSavedPlaylistClicked = async (playlist) => {
      const alert = await alertController.create({
        header: 'Delete Playlist',
        message: `Are you sure you want to delete "${playlist.name}"?`,
        buttons: [
          {
            text: 'Cancel',
            role: 'cancel'
          },
          {
            text: 'Delete',
            role: 'destructive',
            handler: async () => {
              try {
                await apiInstance.delete(`/saved-playlists/${playlist.id}`);
                store.dispatch('app/showToast', {
                  message: 'Playlist deleted',
                  duration: 2000
                });
                loadSavedPlaylists();
              } catch (error) {
                console.error('Failed to delete playlist:', error);
              }
            }
          }
        ]
      });
      await alert.present();
    };

    const onRemoveSavedEntryClicked = async (entry) => {
      try {
        await apiInstance.delete(`/saved-playlists/${viewingPlaylist.value.id}/entries/${entry.id}`);
        // Reload the playlist
        const response = await apiInstance.get(`/saved-playlists/${viewingPlaylist.value.id}`);
        viewingPlaylist.value = response.data;
      } catch (error) {
        console.error('Failed to remove entry:', error);
      }
    };

    const doSavedPlaylistReorder = async (event) => {
      const fromIndex = event.detail.from;
      const toIndex = event.detail.to;

      // Reorder the local array
      const movedItem = viewingPlaylist.value.entries.splice(fromIndex, 1)[0];
      viewingPlaylist.value.entries.splice(toIndex, 0, movedItem);

      try {
        // Update positions on backend
        await apiInstance.put(`/saved-playlists/${viewingPlaylist.value.id}/reorder`, {
          entries: viewingPlaylist.value.entries
        });
        event.detail.complete(true);
      } catch (error) {
        console.error('Failed to reorder playlist:', error);
        // Reload to get correct order
        const response = await apiInstance.get(`/saved-playlists/${viewingPlaylist.value.id}`);
        viewingPlaylist.value = response.data;
        event.detail.complete(false);
      }
    };

    return {
      playerData,
      play,
      add,
      remove,
      trashBin,
      save,
      arrowBack,
      connectedState,
      serverConfigured,
      isPlayerActive,
      doReorder,
      onItemClicked,
      onRemoveItemClicked,
      onClearPlaylistClicked,
      selectedTab,
      savedPlaylists,
      viewingPlaylist,
      onTabChange,
      onSaveCurrentPlaylistClicked,
      onSavedPlaylistClicked,
      onLoadPlaylistClicked,
      onDeleteSavedPlaylistClicked,
      onRemoveSavedEntryClicked,
      doSavedPlaylistReorder,
    };
  },
  components: {
    IonPage,
    IonHeader,
    IonToolbar,
    IonButtons,
    IonMenuButton,
    IonTitle,
    IonContent,
    IonReorder,
    IonReorderGroup,
    IonItem,
    IonLabel,
    IonIcon,
    IonButton,
    IonSegment,
    IonSegmentButton,
    IonList,
    playerController,
  },
};
</script>

<style scoped>
.playlistItemIndicator {
  width: 16px;
  height: 16px;
  margin-right: 5px;
  color: green;
}
</style>
