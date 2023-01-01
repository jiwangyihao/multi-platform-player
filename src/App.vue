<!--suppress SassScssResolvedByNameOnly -->
<script setup lang="ts">
import { computed, ComputedRef, nextTick, Ref, ref } from 'vue';
import { useQuasar } from 'quasar';
import '@chinese-fonts/lxgwwenkai/dist/LXGWWenKai-Bold/result.css';
import Timeout = NodeJS.Timeout;

const state = ref('boot');
const $q = useQuasar();

$q.dark.set(true);
const logo = [ref(), ref(), ref()];
const getLogoWidth = () => {
  logo.forEach((item) => {
    item.value.style.maxWidth = '';
    nextTick(() => {
      item.value.style.maxWidth = `${
        item.value.getBoundingClientRect().width * 1.1
      }px`;
    });
  });
};
nextTick(getLogoWidth);
addEventListener('resize', getLogoWidth);

const isPlay = ref(false);

const nextStep = () => {
  state.value = isPlay.value ? 'player' : 'config';
};

const dragState = ref(false);
const isDragActive = computed(
  () => dragState.value || playList.value.size === 0
);

interface playItem {
  name: string;
  url?: string;
  type: string;
  device?: MediaDeviceInfo;
  ref?: any;
  advance: number;
  volume?: number;
  base?: playItem;
  currentLyric?: lyricsItem;
  interval?: Timeout;
  lyricsContent?: {
    lastTwo: string;
    last: string;
    current: string;
    next: string;
  };
}

interface lyricsItem {
  content: string;
  timeStamp: number;
  lastItem?: lyricsItem;
  nextItem?: lyricsItem;
}

const playList: Ref<Set<playItem>> = ref<Set<playItem>>(new Set());

const listTypeLabel = {
  video: '视频',
  audio: '音乐',
  lyrics: '歌词',
};

const processDrop = async (e: DragEvent) => {
  dragState.value = false;
  const dt = e.dataTransfer;
  const files = dt?.files ?? [];
  for (let i = 0; i < files.length; i++) {
    const file = files[i];

    const fileTypeList = [
      {
        name: 'video/mp4',
        type: 'video',
      },
      {
        name: 'video/webm',
        type: 'video',
      },
      {
        name: 'video/ogg',
        type: 'video',
      },
      {
        name: 'audio/mpeg',
        type: 'audio',
      },
      {
        name: 'audio/x-m4a',
        type: 'audio',
      },
      {
        name: 'audio/wav',
        type: 'audio',
      },
      {
        name: 'audio/flac',
        type: 'audio',
      },
    ];

    let play: playItem | undefined = undefined;

    if (fileTypeList.find((v) => v.name === file.type)) {
      play = {
        name: file.name,
        url: URL.createObjectURL(file),
        type: fileTypeList.find((v) => v.name === file.type)?.type ?? '',
        device: deviceList.value.find((i) => i.value.deviceId === 'default')
          ?.value,
        advance: 0,
        volume: 100,
      };
    } else if (file.name.match(/\.lrc$/gi)) {
      play = {
        name: file.name,
        type: 'lyrics',
        advance: 0,
      };
      const lines = (await file.text()).split('\n');

      let firstItem: lyricsItem;
      let currentItem: lyricsItem;
      lines.forEach((line) => {
        if (line.match(/\[[\d:.]+].+/gi)) {
          const timeArray = (line.match(/(?<=\[).+(?=])/gi) ?? [''])[0].split(
            ':'
          );
          let time = 0;
          for (let j = 0; j < timeArray.length; j++) {
            time +=
              60 ** (timeArray.length - j - 1) *
              Number.parseFloat(timeArray[j]);
          }
          if (!firstItem) {
            firstItem = {
              content: (line.match(/(?<=]).+/gi) ?? [''])[0],
              timeStamp: time,
            };
            currentItem = firstItem;

            (play as playItem).currentLyric = firstItem;
          } else {
            currentItem.nextItem = {
              content: (line.match(/(?<=]).+/gi) ?? [''])[0],
              timeStamp: time,
              lastItem: currentItem,
            };
            currentItem = currentItem.nextItem;
          }
        }
      });

      play.lyricsContent = {
        lastTwo: play.currentLyric?.lastItem?.lastItem?.content ?? '',
        last: play.currentLyric?.lastItem?.content ?? '',
        current: play.currentLyric?.content ?? '',
        next: play.currentLyric?.nextItem?.content ?? '',
      };
      requestAnimationFrame(() => {
        (play?.ref as HTMLDivElement).children[0].innerHTML =
          play?.lyricsContent?.lastTwo ?? '';
        (play?.ref as HTMLDivElement).children[1].innerHTML =
          play?.lyricsContent?.last ?? '';
        (play?.ref as HTMLDivElement).children[2].innerHTML =
          play?.lyricsContent?.current ?? '';
        (play?.ref as HTMLDivElement).children[3].innerHTML =
          play?.lyricsContent?.next ?? '';
      });

      play.interval = setInterval(() => {
        if (play?.base?.ref && play.currentLyric?.nextItem?.timeStamp) {
          if (
            play.base.ref.currentTime + play.advance >
            play.currentLyric.nextItem.timeStamp
          ) {
            play.currentLyric = play.currentLyric.nextItem;
            play.lyricsContent = {
              lastTwo: play.currentLyric?.lastItem?.lastItem?.content ?? '',
              last: play.currentLyric?.lastItem?.content ?? '',
              current: play.currentLyric?.content ?? '',
              next: play.currentLyric?.nextItem?.content ?? '',
            };
            (play?.ref as HTMLDivElement).children[0].innerHTML =
              play?.lyricsContent?.lastTwo ?? '';
            (play?.ref as HTMLDivElement).children[1].innerHTML =
              play?.lyricsContent?.last ?? '';
            (play?.ref as HTMLDivElement).children[2].innerHTML =
              play?.lyricsContent?.current ?? '';
            (play?.ref as HTMLDivElement).children[3].innerHTML =
              play?.lyricsContent?.next ?? '';

            play.ref.classList.remove('animate');

            requestAnimationFrame(() => {
              play.ref.classList.add('animate');
            });
          }
        }
      }, 20);
    }

    if (play) {
      playList.value.add(play);
    }
  }
};

window.addEventListener('dragenter', () => {
  dragState.value = true;
});

interface device {
  label: string;
  value: MediaDeviceInfo;
}

const deviceList = ref<device[]>([]);
const updateDeviceList = async () => {
  const devices = await navigator.mediaDevices.enumerateDevices();
  deviceList.value = devices
    .filter((device) => device.kind === 'audiooutput')
    .map((item) => {
      return { label: item.label, value: item };
    });
};
updateDeviceList();
navigator.mediaDevices.ondevicechange = updateDeviceList;

const beginPlay = async () => {
  isPlay.value = true;
  await $q.fullscreen.request();
  state.value = 'boot';
  for (const item of playList.value) {
    if (['video', 'audio'].includes(item.type)) {
      item.ref.setSinkId(item.device?.deviceId);
      item.ref.currentTime = item.advance / 1000;
      item.ref.volume = (item.volume ?? 100) / 100;
    }
  }
  setTimeout(() => {
    for (const item of playList.value) {
      if (['video', 'audio'].includes(item.type)) {
        item.ref.play();
      }
    }
  }, 2000);
};

const playOption = computed(() => {
  return Array.from(playList.value)
    .map((item) => {
      return {
        label: item.name,
        value: item,
      };
    })
    .filter((item) => ['video', 'audio'].includes(item.value.type));
});
</script>

<template>
  <div
    class="boot row items-center justify-center fixed-full"
    :class="{ active: state === 'boot' }"
  >
    <div class="row" @animationend="nextStep">
      <div><span>M</span><span :ref="logo[0]">ulti-</span></div>
      <div><span>P</span><span :ref="logo[1]">latform</span></div>
      <div><span>P</span><span :ref="logo[2]">layer</span></div>
    </div>
  </div>
  <div class="config fixed-full" :class="{ active: state === 'config' }">
    <q-btn
      v-if="playList.size > 0"
      class="play fixed-bottom"
      icon="play_circle_outline"
      padding="sm"
      rounded
      color="primary"
      size="3vmin"
      @click="beginPlay()"
      style="z-index: 100"
    />
    <div
      class="file-drag fixed-full"
      :class="{ active: isDragActive }"
      @dragenter.stop.prevent
      @dragover.stop.prevent
      @dragleave.stop.prevent="dragState = false"
      @drop.stop.prevent="processDrop"
      style="z-index: 99999"
    >
      <div class="fit row items-center justify-center">
        <h2 class="text-center">请将需要加入队列<br />的文件拖放至此</h2>
      </div>
    </div>
    <q-scroll-area class="list column fit">
      <div class="item" v-for="item in playList" :key="item.name">
        <div class="title row items-center">
          <h6 class="no-margin">{{ item.name }}</h6>
          <q-space />
          <q-badge :label="listTypeLabel[item.type]" color="grey-9" />
        </div>
        <div class="setting row content-between items-center">
          <div
            v-if="['video', 'audio'].includes(item.type)"
            class="choose-device"
          >
            选择播放设备：
            <q-option-group
              v-model="item.device"
              :options="deviceList"
              color="primary"
              inline
              dense
            />
          </div>
          <div class="time-advance row items-center">
            提前量：
            <q-input
              v-model.number="item.advance"
              type="number"
              outlined
              dense
              style="max-width: 100px"
            />
            ms
          </div>
          <div
            v-if="['video', 'audio'].includes(item.type)"
            class="volume row items-center"
            style="width: 200px"
          >
            音量：<q-slider
              v-model="item.volume"
              :min="0"
              :max="100"
              label
              :label-value="item.volume + '%'"
              style="width: auto; flex-grow: 1"
            />
          </div>
          <div v-if="item.type === 'lyrics'" class="volume row items-center">
            选择时间轴基准：
            <q-option-group
              v-model="item.base"
              :options="playOption"
              color="primary"
              inline
              dense
            />
          </div>
        </div>
      </div>
    </q-scroll-area>
  </div>
  <div class="player fixed-full" :class="{ active: state === 'player' }">
    <template v-for="item in playList" :key="item">
      <video
        v-if="item.type === 'video'"
        :src="item.url"
        :ref="
          (el) => {
            item.ref = el;
          }
        "
      ></video>
      <audio
        v-if="item.type === 'audio'"
        :src="item.url"
        :ref="
          (el) => {
            item.ref = el;
          }
        "
      ></audio>
      <div
        v-if="item.type === 'lyrics'"
        class="lyrics-box fixed-center column items-center justify-center"
        :ref="
          (el) => {
            item.ref = el;
          }
        "
      >
        <span></span>
        <span></span>
        <span></span>
        <span></span>
      </div>
    </template>
  </div>
  <div class="control" :class="{ active: state === 'control' }"></div>
</template>

<style lang="scss">
@keyframes scale-in {
  0% {
    transform: scale3d(2, 2, 1);
    opacity: 0;
    filter: blur(1vmin);
  }
  30% {
    transform: scale3d(2, 2, 1);
  }
  50% {
    transform: scale3d(2, 2, 1);
  }
  100% {
    transform: scale3d(1, 1, 1);
  }
}
@keyframes place-color {
  0% {
    color: white;
  }
  50% {
    color: white;
  }
  100% {
    color: $primary;
  }
}
@keyframes width-fill {
  0% {
    max-width: 0;
  }
  50% {
    max-width: 0;
  }
  100% {
  }
}

@keyframes lyric-1 {
  0% {
    transform: scale(0.75) translateY(-120%);
    opacity: 1;
  }
  100% {
    transform: scale(0.5625) translateY(-250%);
  }
}
@keyframes lyric-2 {
  0% {
    transform: unset;
  }
  100% {
    transform: scale(0.75) translateY(-120%);
  }
}
@keyframes lyric-3 {
  0% {
    transform: scale(0.75) translateY(120%);
  }
  100% {
    transform: unset;
  }
}
@keyframes lyric-4 {
  0% {
    transform: scale(0.5625) translateY(250%);
    opacity: 1;
  }
  100% {
    transform: scale(0.75) translateY(120%);
  }
}

* {
  font-family: 'LXGW WenKai', cursive;
}

.boot {
  > div {
    font-size: 6vw;
    justify-content: space-around;
    transform: scale3d(1, 1, 1);

    > div {
      display: flex;
      margin: 0.5vw;

      span:first-child {
        color: $primary;
      }

      > span:last-child {
        display: flex;
        width: auto;
        overflow: hidden;
      }
    }
  }
}
.boot.active {
  > div {
    animation: {
      delay: 1ms;
      duration: 4s;
      name: scale-in;
    }
  }
  > div > div > span:first-child {
    animation: {
      duration: 4s;
      name: place-color;
    }
  }
  > div > div > span:last-child {
    animation: {
      delay: 1ms;
      duration: 4s;
      name: width-fill;
    }
  }
}

.config {
  background-color: rgba(0, 0, 0, 0.2);
  opacity: 0;
  transition: {
    property: opacity;
    duration: 1s;
  }

  &.active {
    opacity: 1;
  }

  .file-drag {
    background-color: rgba(0, 0, 0, 0.5);
    padding: 4vmin;
    opacity: 0;
    visibility: hidden;
    transition: {
      property: opacity, visibility;
      duration: 1s, 0s;
      delay: 0s, 1s;
    }
    &.active {
      opacity: 1;
      visibility: visible;
      transition-delay: 0s;
    }

    * {
      pointer-events: none;
    }

    div {
      border: {
        color: $primary;
        style: solid;
        radius: 4vmin;
        width: 0.5vmin;
      }
    }
  }

  .list {
    padding: 2vmin;

    .item {
      margin: 2vmin;
      border: {
        color: #666;
        style: solid;
        radius: 2vmin;
        width: 0.5vmin;
      }
      background-color: rgba(0, 0, 0, 0.5);
      padding: 1vmin;
      transition: {
        property: border-color;
        duration: 0.5s;
      }

      &:hover {
        border-color: $primary;
      }

      .setting > * {
        margin: 1vmin;
      }
    }
  }

  .play {
    width: fit-content;
    margin: 0 auto;
    bottom: 4vmin;
  }
}

.player {
  background-color: rgba(0, 0, 0, 0.2);
  opacity: 0;
  transition: {
    property: opacity;
    duration: 1s;
  }
  pointer-events: none;

  &.active {
    opacity: 1;
  }

  video {
    width: 100%;
    height: 100%;
  }

  .lyrics-box {
    height: 20vh;
    width: 100%;
    background-color: rgba(0, 0, 0, 0.5);

    > span {
      position: absolute;
      white-space: nowrap;
      font-size: 6vh;
      line-height: 6vh;

      &:nth-child(1) {
        transform: scale(0.5625) translateY(-250%);
        opacity: 0;
      }

      &:nth-child(2) {
        transform: scale(0.75) translateY(-120%);
      }

      &:nth-child(4) {
        transform: scale(0.75) translateY(120%);
      }
    }

    &.animate {
      > span {
        &:nth-child(1) {
          animation: {
            duration: 0.5s;
            name: lyric-1;
          }
        }
        &:nth-child(2) {
          animation: {
            duration: 0.5s;
            name: lyric-2;
          }
        }
        &:nth-child(3) {
          animation: {
            duration: 0.5s;
            name: lyric-3;
          }
        }
        &:nth-child(4) {
          animation: {
            duration: 0.5s;
            name: lyric-4;
          }
        }
      }
    }
  }
}
</style>
