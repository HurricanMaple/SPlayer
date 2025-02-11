<template>
  <Transition name="up">
    <div
      v-show="music.showBigPlayer"
      class="bplayer"
      :style="
        music.getPlaySongData
          ? 'background-image: url(' +
            music.getPlaySongData.album.picUrl.replace(/^http:/, 'https:') +
            '?param=50y50)'
          : ''
      "
    >
      <div class="gray" />
      <n-icon
        class="close"
        size="40"
        :component="KeyboardArrowDownFilled"
        @click="music.setBigPlayerState(false)"
      />
      <div :class="music.getPlaySongLyric[0] ? 'all' : 'all noLrc'">
        <div class="left">
          <PlayerCover v-if="setting.playerStyle === 'cover'" />
          <PlayerRecord v-else />
        </div>
        <div
          class="right"
          @mouseenter="menuShow = true"
          @mouseleave="menuShow = false"
        >
          <Transition name="lrc">
            <div class="lrcShow" v-if="music.getPlaySongLyric[0]">
              <div class="data" v-show="setting.playerStyle === 'record'">
                <div class="name text-hidden">
                  <span>{{
                    music.getPlaySongData
                      ? music.getPlaySongData.name
                      : "暂无歌曲"
                  }}</span>
                  <span
                    v-if="music.getPlaySongData && music.getPlaySongData.alia"
                    >{{ music.getPlaySongData.alia[0] }}</span
                  >
                </div>
                <div
                  class="artists text-hidden"
                  v-if="music.getPlaySongData && music.getPlaySongData.artist"
                >
                  <span
                    class="artist"
                    v-for="(item, index) in music.getPlaySongData.artist"
                    :key="item"
                  >
                    <span>{{ item.name }}</span>
                    <span
                      v-if="index != music.getPlaySongData.artist.length - 1"
                      >/</span
                    >
                  </span>
                </div>
              </div>
              <div
                :class="
                  setting.playerStyle === 'cover'
                    ? 'lrc-all cover'
                    : 'lrc-all record'
                "
                v-if="music.getPlaySongLyric[0]"
                :style="
                  setting.lyricsPosition === 'center'
                    ? 'text-align: center'
                    : null
                "
              >
                <div class="placeholder"></div>
                <div
                  :class="lrcOnPlayIndex == index ? 'lrc on' : 'lrc'"
                  v-for="(item, index) in music.getPlaySongLyric"
                  :key="item"
                  :id="'lrc' + index"
                  @click="jumpTime(item.time)"
                >
                  <span class="lyric">{{ item.lyric }}</span>
                  <span
                    v-show="
                      music.getPlaySongTransl &&
                      setting.getShowTransl &&
                      item.lyricFy
                    "
                    class="lyric-fy"
                    >{{ item.lyricFy }}</span
                  >
                </div>
                <div class="placeholder"></div>
              </div>
              <div
                :class="menuShow ? 'menu show' : 'menu'"
                v-show="setting.playerStyle === 'record'"
              >
                <n-icon
                  v-if="music.getPlaySongTransl"
                  :class="setting.getShowTransl ? 'open' : ''"
                  :component="GTranslateFilled"
                  @click="setting.setShowTransl(!setting.getShowTransl)"
                />
                <n-icon
                  class="open"
                  :component="MessageFilled"
                  @click="toComment"
                />
              </div>
            </div>
          </Transition>
        </div>
      </div>
      <canvas v-if="setting.musicFrequency" class="avBars" ref="avBars" />
    </div>
  </Transition>
</template>

<script setup>
import {
  KeyboardArrowDownFilled,
  GTranslateFilled,
  MessageFilled,
} from "@vicons/material";
import { musicStore, settingStore } from "@/store/index";
import { useRouter } from "vue-router";
import MusicFrequency from "@/utils/MusicFrequency.js";
import PlayerRecord from "./PlayerRecord.vue";
import PlayerCover from "./PlayerCover.vue";

const router = useRouter();
const music = musicStore();
const setting = settingStore();

// 工具栏显隐
let menuShow = ref(false);

// 歌词数据
let lrcOnPlayIndex = ref(0);
let lrcInterval = ref(null);

// 音乐频谱
let avBars = ref(null);
let musicFrequency = ref(null);

// 点击歌词跳转
const jumpTime = (time) => {
  if ($player) $player.currentTime = time;
};

// 前往评论
const toComment = () => {
  music.setBigPlayerState(false);
  router.push({
    path: "/comment",
    query: {
      id: music.getPlaySongData ? music.getPlaySongData.id : null,
    },
  });
};

// 歌词滚动
const lyricsScroll = () => {
  lrcInterval.value = setInterval(() => {
    // console.log(11);
    let oldIndex = lrcOnPlayIndex.value;
    let index = music.getPlaySongLyric.findIndex(
      (v) => v.time > music.getPlaySongTime.currentTime
    );
    if (index === -1) {
      // 如果没有找到合适的歌词，则返回最后一句歌词
      lrcOnPlayIndex.value = music.getPlaySongLyric.length - 1;
      music.lrcOnPlayIndex = music.getPlaySongLyric.length - 1;
    } else {
      lrcOnPlayIndex.value = index - 1;
      music.lrcOnPlayIndex = index - 1;
    }
    if (oldIndex !== lrcOnPlayIndex.value) {
      const el = document.getElementById(`lrc${lrcOnPlayIndex.value}`);
      if (el) {
        el.scrollIntoView({
          behavior: "smooth",
          block: "center",
        });
      }
    }
  }, 500);
};

onMounted(() => {
  nextTick(() => {
    if (setting.musicFrequency) {
      $player.crossOrigin = "anonymous";
      musicFrequency.value = new MusicFrequency(
        avBars.value,
        $player,
        null,
        50,
        null,
        null,
        5
      );
      musicFrequency.value.drawSpectrum();
    }
  });
});

// 监听页面是否打开
watch(
  () => music.showBigPlayer,
  (val) => {
    if (val) {
      clearInterval(lrcInterval.value);
      lyricsScroll();
    } else {
      clearInterval(lrcInterval.value);
    }
  }
);

// 音乐频谱适配
// watch(
//   () => music.getPlaySongData,
//   () => {
//     if (setting.musicFrequency) $player.play();
//   }
// );

onBeforeUnmount(() => {
  clearInterval(lrcInterval.value);
});
</script>

<style lang="scss" scoped>
.up-enter-active,
.up-leave-active {
  transform: translateY(0);
  transition: all 0.5s cubic-bezier(0.65, 0.05, 0.36, 1);
}
.up-enter-from,
.up-leave-to {
  transform: translateY(100%);
}
.lrc-enter-active,
.lrc-leave-active {
  transition: opacity 0.3s ease;
}
.lrc-enter-active {
  transition-delay: 0.3s;
}
.lrc-enter-from,
.lrc-leave-to {
  opacity: 0;
}
.bplayer {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 9999;
  overflow: hidden;
  color: #ffffff;
  background-repeat: no-repeat;
  background-size: 150% 150%;
  background-position: center;
  display: flex;
  justify-content: center;
  &::after {
    // content: "";
    position: absolute;
    top: 0;
    left: calc(50% - 2px);
    height: 100%;
    width: 4px;
    background-color: red;
  }
  .gray {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: #00000060;
    backdrop-filter: blur(80px);
    z-index: -1;
  }
  .close {
    position: absolute;
    top: 24px;
    right: 24px;
    opacity: 0.3;
    color: #fff;
    cursor: pointer;
    border-radius: 8px;
    transition: all 0.3s;
    &:hover {
      background-color: #ffffff20;
      transform: scale(1.05);
      opacity: 1;
    }
    &:active {
      transform: scale(1);
    }
  }
  .all {
    width: 100%;
    height: 100%;
    display: flex;
    flex-direction: row;
    align-items: center;
    transition: all 0.3s ease-in-out;
    &.noLrc {
      .left {
        transform: translateX(25vh);
      }
      @media (max-width: 768px) {
        .left {
          display: flex !important;
          transform: none;
          align-items: center;
        }
        .right {
          display: none !important;
        }
      }
    }
    @media (max-width: 768px) {
      .left {
        display: none !important;
      }
      .right {
        padding: 0 5vw;
        .lrcShow {
          .lrc-all {
            height: 70vh !important;
            padding-right: 0 !important;
          }
          .data,
          .menu {
            display: block !important;
            opacity: 1 !important;
          }
        }
      }
    }
    .left {
      flex: 1;
      padding: 0 4vw;
      display: flex;
      flex-direction: column;
      align-items: flex-end;
      justify-content: center;
      transition: all 0.3s ease-in-out;
    }
    .right {
      flex: 1;
      height: 100%;
      // padding: 0 4vw;
      .lrcShow {
        height: 100%;
        display: flex;
        justify-content: center;
        flex-direction: column;
        .data {
          padding: 0 20px;
          margin-bottom: 8px;
          .name {
            font-size: 3vh;
            -webkit-line-clamp: 2;
            padding-right: 26px;
            span {
              &:nth-of-type(2) {
                margin-left: 12px;
                font-size: 2.3vh;
                opacity: 0.6;
              }
            }
          }
          .artists {
            margin-top: 4px;
            opacity: 0.6;
            font-size: 1.8vh;
            .artist {
              span {
                &:nth-of-type(2) {
                  margin: 0 2px;
                }
              }
            }
          }
        }
        .lrc-all {
          padding-right: 14%;
          scrollbar-width: none;
          max-width: 460px;
          overflow: auto;
          mask: linear-gradient(
            180deg,
            hsla(0, 0%, 100%, 0) 0,
            hsla(0, 0%, 100%, 0.6) 15%,
            #fff 25%,
            #fff 75%,
            hsla(0, 0%, 100%, 0.6) 85%,
            hsla(0, 0%, 100%, 0)
          );
          -webkit-mask: linear-gradient(
            180deg,
            hsla(0, 0%, 100%, 0) 0,
            hsla(0, 0%, 100%, 0.6) 15%,
            #fff 25%,
            #fff 75%,
            hsla(0, 0%, 100%, 0.6) 85%,
            hsla(0, 0%, 100%, 0)
          );
          &::-webkit-scrollbar {
            display: none;
          }
          &.cover {
            height: 80vh;
          }
          &.record {
            height: 60vh;
          }
          .placeholder {
            width: 100%;
            height: 50%;
          }
          .lrc {
            opacity: 0.6;
            transition: all 0.3s;
            display: flex;
            flex-direction: column;
            // margin-bottom: 4px;
            // padding: 12px 20px;
            margin-bottom: 0.8vh;
            padding: 1.8vh 3vh;
            border-radius: 8px;
            transition: all 0.3s;
            cursor: pointer;
            .lyric {
              transition: all 0.3s;
              font-size: 2.3vh;
            }
            .lyric-fy {
              margin-top: 2px;
              transition: all 0.3s;
              opacity: 0.8;
              font-size: 2vh;
            }
            &.on {
              opacity: 1;
              .lyric {
                font-weight: bold;
                font-size: 3vh;
              }
              .lyric-fy {
                font-weight: normal;
                font-size: 2.3vh;
              }
            }
            &:hover {
              @media (min-width: 768px) {
                background-color: #ffffff20;
              }
            }
            &:active {
              transform: scale(0.95);
            }
          }
        }
        .menu {
          opacity: 0;
          padding: 0 20px;
          margin-top: 20px;
          display: flex;
          flex-direction: row;
          align-items: center;
          transition: all 0.3s;
          &.show {
            opacity: 1;
          }
          .n-icon {
            margin-right: 8px;
            font-size: 24px;
            cursor: pointer;
            padding: 8px;
            border-radius: 8px;
            opacity: 0.4;
            transition: all 0.3s;
            &:hover {
              background-color: #ffffff30;
            }
            &:active {
              transform: scale(0.95);
            }
            &.open {
              opacity: 1;
            }
          }
        }
      }
    }
  }
  .avBars {
    z-index: -1;
    position: absolute;
    bottom: 0;
    opacity: 0.6;
    -webkit-mask: linear-gradient(
      to right,
      hsla(0deg, 0%, 100%, 0) 0,
      hsla(0deg, 0%, 100%, 0.6) 15%,
      #fff 30%,
      #fff 70%,
      hsla(0deg, 0%, 100%, 0.6) 85%,
      hsla(0deg, 0%, 100%, 0)
    );
    mask: linear-gradient(
      to right,
      hsla(0deg, 0%, 100%, 0) 0,
      hsla(0deg, 0%, 100%, 0.6) 15%,
      #fff 30%,
      #fff 70%,
      hsla(0deg, 0%, 100%, 0.6) 85%,
      hsla(0deg, 0%, 100%, 0)
    );
  }
}
</style>