<template>
  <ion-footer>
    <ion-row class="favoriteRow">
      <ion-col size="12">
        <ion-button @click="onFavoriteClicked" fill="clear" size="small" class="favoriteButton">
          Favorite {{ mediaType }}?
          <ion-icon
            slot="end"
            :icon="isFavorited ? star : starOutline"
            :class="{ favorited: isFavorited }"
          ></ion-icon>
        </ion-button>
      </ion-col>
    </ion-row>
    <ion-row class="ion-justify-content-end">
      <ion-col size="12" class="videotitle">
        <div class="playbackTime">
          {{ formatTime(playerData.position) }} /
          {{ formatTime(playerData.duration) }}
        </div>
        <input
          id="seekbar"
          type="range"
          name="rng"
          min="0"
          step="1"
          :max="playerData.duration"
          :value="playerData.position"
          @input="onSeek"
        />
      </ion-col>
    </ion-row>
    <ion-row class="videoControls">
      <ion-col size="12">
        <ion-button @click="onPlayPauseClicked" fill="clear">
          <ion-icon
            v-if="playerData.pause"
            slot="icon-only"
            :icon="playOutline"
          ></ion-icon>
          <ion-icon v-else slot="icon-only" :icon="pauseOutline"></ion-icon>
        </ion-button>
        <ion-button @click="onStopClicked" fill="clear">
          <ion-icon slot="icon-only" :icon="stopOutline"></ion-icon>
        </ion-button>
        <ion-button @click="onPrevClicked" fill="clear">
          <ion-icon slot="icon-only" :icon="playSkipBackOutline"></ion-icon>
        </ion-button>
        <ion-button @click="onSkip(-5)" fill="clear"> -5 </ion-button>
        <ion-button @click="onSkip(5)" fill="clear"> +5 </ion-button>
        <ion-button @click="onNextClicked" fill="clear">
          <ion-icon slot="icon-only" :icon="playSkipForwardOutline"></ion-icon>
        </ion-button>
      </ion-col>
    </ion-row>
  </ion-footer>
</template>

<script>
import { computed, ref, watch } from "vue";
import { useStore } from "vuex";
import musicControls from "cordova-plugin-music-controls2/www/MusicControls.js";
import { IonFooter, IonRow, IonCol, IonIcon, IonButton } from "@ionic/vue";
import {
  playOutline,
  pauseOutline,
  stopOutline,
  playSkipBackOutline,
  playSkipForwardOutline,
  star,
  starOutline,
} from "ionicons/icons";
import { formatTime, seekFlags } from "../tools";
import { apiInstance } from "../api";
export default {
  setup() {
    const store = useStore();
    const playerData = computed(() => store.state.simpleapi.playerData);
    const isFavorited = ref(false);

    // Determine if current media is audio or video
    const mediaType = computed(() => {
      // Check if there's a video track
      if (playerData.value['track-list']) {
        const hasVideo = playerData.value['track-list'].some(
          track => track.type === 'video'
        );
        return hasVideo ? 'Video' : 'Song';
      }
      // Fallback: check by file extension
      if (playerData.value.filename) {
        const ext = playerData.value.filename.split('.').pop().toLowerCase();
        const audioExtensions = ['mp3', 'flac', 'wav', 'ogg', 'aac', 'm4a', 'wma', 'opus'];
        return audioExtensions.includes(ext) ? 'Song' : 'Video';
      }
      return 'Video'; // default
    });

    // Watch for path changes to update favorite status
    watch(() => playerData.value.path, async (newPath) => {
      if (newPath) {
        try {
          const response = await apiInstance.get(`/favorites/status/${encodeURIComponent(newPath)}`);
          isFavorited.value = response.data.favorited === 1;
        } catch (error) {
          console.error("Failed to fetch favorite status:", error);
          isFavorited.value = false;
        }
      } else {
        isFavorited.value = false;
      }
    }, { immediate: true });

    const onPlayPauseClicked = () => {
      apiInstance.post("/controls/play-pause").then((response) => {
        if (response.data.message == "success") {
          store.commit("simpleapi/setPlayerDataProperty", {
            key: "pause",
            value: !playerData.value.pause,
          });
          if (store.getters["settings/androidNotificationEnabled"] == true)
            musicControls.updateIsPlaying(!playerData.value.pause);
        }
      });
    };

    const onSeek = (e) => {
      apiInstance
        .post("controls/seek", {
          target: e.target.value,
          flag: seekFlags.ABSOLUTE,
        })
        .then(() => {
          store.commit("simpleapi/setPlayerDataProperty", {
            key: "position",
            value: e.target.value,
          });
        });
    };

    const onStopClicked = () => {
      apiInstance.post("controls/stop");
    };

    const onPrevClicked = () => {
      apiInstance.post("controls/prev");
    };

    const onNextClicked = () => {
      apiInstance.post("controls/next");
    };

    const onSkip = (seconds) => {
      apiInstance
        .post("controls/seek", {
          target: seconds,
          flag: seekFlags.RELATIVE,
        })
        .then(() => {
          store.commit("simpleapi/setPlayerDataProperty", {
            key: "position",
            value: playerData.value.position + seconds,
          });
        });
    };

    const onFavoriteClicked = async () => {
      if (!playerData.value.path) return;

      try {
        const response = await apiInstance.post("/favorites/toggle", {
          filepath: playerData.value.path,
        });
        isFavorited.value = response.data.favorited === 1;

        store.dispatch("app/showToast", {
          message: isFavorited.value ? "Added to favorites" : "Removed from favorites",
          duration: 1500,
        });
      } catch (error) {
        console.error("Failed to toggle favorite:", error);
        store.dispatch("app/showToast", {
          message: "Failed to update favorite status",
          duration: 2000,
        });
      }
    };

    return {
      playerData,
      onPlayPauseClicked,
      onStopClicked,
      onPrevClicked,
      onNextClicked,
      onSeek,
      onSkip,
      onFavoriteClicked,
      isFavorited,
      mediaType,
      playOutline,
      pauseOutline,
      stopOutline,
      playSkipBackOutline,
      playSkipForwardOutline,
      star,
      starOutline,
      formatTime,
    };
  },
  components: {
    IonFooter,
    IonRow,
    IonCol,
    IonIcon,
    IonButton,
  },
};
</script>

<style scoped>
.favoriteRow {
  text-align: center;
  padding-top: 5px;
}

.favoriteButton {
  color: #fff;
  font-size: 14px;
}

.favoriteButton ion-icon {
  font-size: 24px;
  margin-left: 5px;
  stroke-width: 0;
  position: relative;
  top: -2px;
}

.favoriteButton ion-icon.favorited {
  color: #ffd700;
}

.videoControls {
  text-align: center;
  padding: 0 5px;
}
.videoControls ion-col {
  display: flex;
  justify-content: center;
  gap: 2px;
}
.videoControls ion-button {
  color: #fff;
  font-size: 20px;
  min-width: 70px;
  height: 70px;
  flex: 1;
  max-width: 150px;
  margin: 0;
}

.videotitle {
  padding: 10px;
}

.playbackTime {
  text-align: center;
  align-content: center;
}

#seekbar {
  width: 100%;
  height: 5px;
}

/*
Seekbar style
*/
input[type="range"] {
  -webkit-appearance: none;
  width: 80%;
}
input[type="range"]:focus {
  outline: none;
}
input[type="range"]::-webkit-slider-runnable-track {
  width: 100%;
  height: 5px;
  cursor: pointer;
  box-shadow: 1px 1px 1px #000000, 0px 0px 1px #0d0d0d;
  background: #455795;
  border-radius: 1px;
  border: 0.1px solid #010101;
}
input[type="range"]::-webkit-slider-thumb {
  box-shadow: 1px 1px 1px #000000, 0px 0px 1px #0d0d0d;
  border: 1px solid #000000;
  height: 18px;
  width: 13px;
  border-radius: 3px;
  background: #26335c;
  cursor: pointer;
  -webkit-appearance: none;
  margin-top: -7px;
}

input[type="range"]:focus::-webkit-slider-runnable-track {
  background: #367ebd;
}
input[type="range"]::-moz-range-track {
  width: 100%;
  height: 5px;
  cursor: pointer;
  box-shadow: 1px 1px 1px #000000, 0px 0px 1px #0d0d0d;
  background: #455795;
  border-radius: 1px;
  border: 0.1px solid #010101;
}
input[type="range"]::-moz-range-thumb {
  box-shadow: 1px 1px 1px #000000, 0px 0px 1px #0d0d0d;
  border: 1px solid #000000;
  height: 18px;
  width: 13px;
  border-radius: 3px;
  background: #26335c;
  cursor: pointer;
}
input[type="range"]::-ms-track {
  width: 100%;
  height: 8.4px;
  cursor: pointer;
  background: transparent;
  border-color: transparent;
  border-width: 16px 0;
  color: transparent;
}
input[type="range"]::-ms-fill-lower {
  background: #2a6495;
  border: 0.2px solid #010101;
  border-radius: 2.6px;
  box-shadow: 1px 1px 1px #000000, 0px 0px 1px #0d0d0d;
}
input[type="range"]::-ms-fill-upper {
  background: #3071a9;
  border: 0.2px solid #010101;
  border-radius: 2.6px;
  box-shadow: 1px 1px 1px #000000, 0px 0px 1px #0d0d0d;
}
input[type="range"]::-ms-thumb {
  box-shadow: 1px 1px 1px #000000, 0px 0px 1px #0d0d0d;
  border: 1px solid #000000;
  height: 36px;
  width: 16px;
  border-radius: 3px;
  background: #26335c;
  cursor: pointer;
}
input[type="range"]:focus::-ms-fill-lower {
  background: #3071a9;
}
input[type="range"]:focus::-ms-fill-upper {
  background: #367ebd;
}

@media (min-width: 576px) and (orientation: landscape) {
  .playbackTime {
    display: none;
  }
}
</style>
