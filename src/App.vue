<template>
  <main id="app">
    <div class="scroll-body">
      <div class="grid-wrapper" :class="{ 'move-active': isMoveActive }">
        <template v-if="current">
          <div
            class="grid-body"
            :class="layoutData.className"
            @dragstart="dragStart"
            @dragend="dragEnd"
            @dragover.prevent="$event.dataTransfer.dropEffect = 'move'"
            @dragenter="dragHover"
            @dragleave="dragHover"
            @drop.prevent="dropColumn"
          >
            <div class="grid-column" v-for="n of layoutData.n" :key="n.id">
              <div class="column-inner" :data-index="n">
                <div v-if="videoIds[n]" class="frame-wrapper">
                  <youtube :video-id="videoIds[n]" :player-vars="{ fs: 0 }" :ref="`youtube-${n}`" @playing="playing"></youtube>
                </div>
                <div v-else class="empty"></div>

                <div class="url-block" :ref="`url-${n}`">
                  <p>動画・配信のURLを入力</p>
                  <input
                    type="url"
                    placeholder="https://www.youtube.com/watch?v=..."
                    pattern="^https://(www\.youtube\.com/watch\?v=|youtu\.be/).+"
                    required
                    :ref="`input-${n}`"
                  >
                  <div class="button-wrapper">
                    <button @click="setVideoId(n)" class="btn-ok">OK</button>
                    <button v-if="videoIds[n]" @click="deleteVideoId(n)" class="btn-reset">リセット</button>
                  </div>
                </div>

                <div class="move-block" draggable="true"></div>
              </div>
            </div>
          </div>
        </template>

        <div v-else class="start-block">
          <h1>Multi YouTube</h1>

          <p>YouTubeの動画や配信をひとつのウィンドウに複数表示させるだけのツールです。</p>

          <div class="button-wrapper">
            <button class="layout" @click="showLayout">
              <img src="./assets/icon_layout.svg" alt="Layout">
              レイアウトを選ぶ
            </button>
          </div>
        </div>
      </div>

      <div class="select-wrapper" :class="{ show: isSelectLayout }">
        <div class="select-body">
          <p>レイアウトを選んでください</p>
          <ul class="select-layout">
            <li v-for="[name, layout] of layouts" :key="layout.id">
              <input type="radio" v-model="current" :id="name" :value="name">
              <label :for="name">
                <div :class="name"><span v-for="n of layout.n" :key="n.id"></span></div>
              </label>
            </li>
          </ul>
          <div class="button-wrapper">
            <button class="btn-ok" @click="setLayout" :disabled="!current">OK</button>
          </div>
        </div>
      </div>

      <nav class="controller-wapper" v-if="current">
        <ul class="controller">
          <li :class="{ 'is-ignore': isMoveActive }">
            <button @click="control('pauseVideo')">
              <img src="./assets/icon_pause.svg" alt="Pause">
            </button>
            <span class="tooltip">まとめて一時停止</span>
          </li>
          <li :class="{ 'is-ignore': isMoveActive }">
            <button @click="control('unMute')">
              <img src="./assets/icon_speaker.svg" alt="Unmute">
            </button>
            <span class="tooltip">まとめてミュート解除</span>
          </li>
          <li :class="{ 'is-ignore': isMoveActive }">
            <button @click="control('mute')">
              <img src="./assets/icon_mute.svg" alt="Mute">
            </button>
            <span class="tooltip">まとめてミュート</span>
          </li>
          <li v-if="enabled" class="sp-none">
            <template v-if="isFullScreen">
              <button @click="fullscreen(false)">
                <img src="./assets/icon_full_exit.svg" alt="Exit full screen">
              </button>
              <span class="tooltip">フルスクリーン解除</span>
            </template>
            <template v-else>
              <button @click="fullscreen(true)">
                <img src="./assets/icon_full.svg" alt="Full screen">
              </button>
              <span class="tooltip">フルスクリーン表示</span>
            </template>
          </li>
          <li>
            <button @click="showURL">
              <img src="./assets/icon_gear.svg" alt="Change URL">
            </button>
            <span class="tooltip">動画・配信の変更</span>
          </li>
          <li class="sp-none">
            <button @click="toggleMove">
              <img src="./assets/icon_move.svg" alt="Change layout">
            </button>
            <span class="tooltip">配置の変更</span>
          </li>
          <li>
            <button @click="showLayout">
              <img src="./assets/icon_layout.svg" alt="Change layout">
            </button>
            <span class="tooltip">レイアウト変更</span>
          </li>
          <li>
            <button @click="allReset">
              <img src="./assets/icon_delete.svg" alt="Reset">
            </button>
            <span class="tooltip">リセット</span>
          </li>
        </ul>
      </nav>

      <aside v-if="isMoveActive" class="move-float">
        <button @click="toggleMove">配置の変更を終了</button>
      </aside>
    </div>
  </main>
</template>

<script>
import Vue from 'vue'
import VueYoutube from 'vue-youtube'

import SimpleBar from 'simplebar'
import 'simplebar/dist/simplebar.css'

Vue.use(VueYoutube)

/** Video id のチェック用 */
const VIDEO_ID_REGEX = /^[-\w]{11,12}$/

/**
 * 動画IDマスター
 * @type {Map<number, string>}
 */
const VIDEO_ID_MAP = new Map

/**
 * スクロール要素
 * @type {HTMLElement}
 */
let scrollElement

/** ハッシュ変更時にイベントを実行するかのフラグ */
let isHashChange = true

/**
 * Map をハッシュ用文字列へ変換
 * @param {Map<number, string>} map
 */
const mapToStr = map => [...map].map(v => v.join(':')).join()

export default {
  name: 'App',

  data() {
    return {
      /**
       * 現在のレイアウト
       * @type {?string}
       */
      current: null,

      /**
       * レイアウト情報
       * @type {Map<string, { n: number, className: string[] }>}
       */
      layouts: new Map([
        ['l-1', { n: 1, className: ['grid-layout-1x1'] }],
        ['l-2', { n: 2, className: ['grid-layout-1x2'] }],
        ['l-3', { n: 2, className: ['grid-layout-2x1'] }],
        ['l-4', { n: 3, className: ['grid-layout-4x4', 'grid-column-2-t'] }],
        ['l-5', { n: 3, className: ['grid-layout-4x4', 'grid-column-2-b'] }],
        ['l-6', { n: 4, className: ['grid-layout-4x4', 'grid-column-2x2'] }],
        ['l-7', { n: 6, className: ['grid-layout-2x3'] }],
        ['l-8', { n: 6, className: ['grid-layout-3x2'] }],
        ['l-9', { n: 9, className: ['grid-layout-6x6', 'grid-column-3x3'] }],
        ['l-10', { n: 3, className: ['grid-layout-2x3', 'grid-column-large'] }],
        ['l-11', { n: 3, className: ['grid-layout-3x2', 'grid-column-large'] }],
        ['l-12', { n: 6, className: ['grid-layout-6x6', 'grid-column-large'] }]
      ]),

      /**
       * 再生中のプレイヤー
       * @type {Player}
       */
      currentPlayer: null,

      /**
       * YouTubeのIDマップ
       * @type {Object<number, string>}
       */
      videoIds: {},

      /** レイアウト選択フラグ */
      isSelectLayout: false,

      /** フルスクリーンフラグ */
      isFullScreen: false,

      /** 配置変更モードフラグ */
      isMoveActive: false,

      /** フルスクリーンが使用可能か */
      enabled: !!document.fullscreenEnabled,

      /**
       * ドラッグ中の要素
       * @type {?HTMLElement}
       */
      dragged: null
    }
  },

  computed: {
    /**
     * 選択されたレイアウトデータの取得
     * @return {{ n: number, className: string[] }}
     */
    layoutData() {
      return this.layouts.get(this.current)
    }
  },

  methods: {
    /**
     * 特定の要素の取得
     * @param {string} name
     * @return {HTMLElement}
     */
    getRefs(name) {
      return this.$refs[name][0]
    },

    /** レイアウト選択画面の表示 */
    showLayout() {
      this.isMoveActive = false
      this.isSelectLayout = true
      scrollElement.scrollTop = 0
      this.control('pauseVideo')
    },

    /** レイアウトの適用 */
    setLayout() {
      const hash = location.hash.slice(2)
      location.hash = hash ? '/' + hash.replace(/^l-\d+/, this.current) : `/${this.current}/`
      this.isSelectLayout = false
    },

    /**
     * YouTubeプレイヤーのセット
     * @param {number} n
     */
    setVideoId(n) {
      /** @type {string} */
      const url = this.getRefs(`input-${n}`).value

      /** @type {string} */
      const videoId = this.$youtube.getIdFromUrl(url)

      if (VIDEO_ID_REGEX.test(videoId)) {
        isHashChange = false
        VIDEO_ID_MAP.set(n, videoId)
        this.getRefs(`url-${n}`).classList.add('hidden')
        location.hash = `/${this.current}/` + mapToStr(VIDEO_ID_MAP)
      }
    },

    /**
     * YouTubeプレイヤーの削除
     * @param {number} n
     */
    deleteVideoId(n) {
      isHashChange = false
      VIDEO_ID_MAP.delete(n)
      this.getRefs(`input-${n}`).value = ''
      location.hash = `/${this.current}/` + mapToStr(VIDEO_ID_MAP)
    },

    /**
     * プレイヤーの操作
     * @param {string} method
     */
    control(method) {
      for (const [n] of VIDEO_ID_MAP) {
        this.getRefs(`youtube-${n}`).player[method]()

        if (method === 'playVideo') {
          this.getRefs(`url-${n}`).classList.add('hidden')
        }
      }
    },

    /**
     * フルスクリーン操作
     * @param {boolean} bool
     */
    async fullscreen(bool) {
      if (bool) {
        await this.$el.requestFullscreen()
        scrollElement.scrollTop = 0
      } else {
        document.exitFullscreen()
      }
    },

    /** 配置変更 */
    toggleMove() {
      this.isMoveActive = !this.isMoveActive

      if (this.isMoveActive) {
        this.control('pauseVideo')
      }
    },

    /** URL設定画面の表示 */
    showURL() {
      this.isMoveActive = false
      this.control('pauseVideo')

      const { n } = this.layoutData
      for (let i = 1; i <= n; i++) {
        this.getRefs(`url-${i}`).classList.remove('hidden')
      }
    },

    /** 全てをリセット */
    allReset() {
      this.current = null
      this.videoIds = {}
      this.isMoveActive = false
      VIDEO_ID_MAP.clear()
      history.pushState(null, null, './')
    },

    /**
     * ドラッグスタート
     * @param {DragEvent} event
     */
    dragStart(event) {
      event.dataTransfer.dropEffect = 'move'
      this.dragged = event.target.closest('.column-inner')
      this.dragged.classList.add('dragged')
    },

    /** ドラッグおわり */
    dragEnd() {
      this.dragged?.classList.remove('dragged')
    },

    /**
     * ドラッグ中の重なり
     * @param {DragEvent} event
     */
    dragHover(event) {
      /** @type {?HTMLElement} */
      const elem = event.target.closest('.column-inner')
      elem?.classList.toggle('drag-enter', event.type === 'dragenter')
    },

    /**
     * 要素入れ替え
     * @param {DragEvent} event
     */
    dropColumn(event) {
      /** @type {HTMLElement} */
      const target = event.target.closest('.column-inner')

      this.dragged.classList.remove('drag-enter', 'dragged')

      if (target && target !== this.dragged) {
        target.classList.remove('drag-enter')

        // 入れ替える要素のデータ
        const draggedIndex = Number(this.dragged.dataset.index)
        const draggedData = VIDEO_ID_MAP.get(draggedIndex)

        // 入れ替えられる要素のデータ
        const targetIndex = Number(target.dataset.index)
        const targetData = VIDEO_ID_MAP.get(targetIndex)

        if (draggedData) {
          VIDEO_ID_MAP.set(targetIndex, draggedData)
        } else {
          VIDEO_ID_MAP.delete(targetIndex)
          this.getRefs(`input-${targetIndex}`).value = ''
        }

        if (targetData) {
          VIDEO_ID_MAP.set(draggedIndex, targetData)
        } else {
          VIDEO_ID_MAP.delete(draggedIndex)
          this.getRefs(`input-${draggedIndex}`).value = ''
        }

        location.hash = `/${this.current}/` + mapToStr(VIDEO_ID_MAP)
      }

      this.dragged = null
    },

    /**
     * 再生監視
     * @param {Player} player
     */
    async playing(player) {
      await this.currentPlayer?.pauseVideo()
      this.currentPlayer = player
    }
  },

  mounted() {
    /** ハッシュ監視 */
    const onHashChange = () => {
      const [layout, ids = ''] = location.hash.slice(2).split('/')

      VIDEO_ID_MAP.clear()

      // 値がおかしい場合リセット
      if (!layout || !this.layouts.has(layout)) {
        this.current = null
        this.videoIds = {}
        history.replaceState(null, null, './')
        return
      }

      this.isSelectLayout = false
      this.current = layout

      const { n } = this.layouts.get(layout)

      /** @type {[string, string][]} */
      const idMap = ids ? ids.split(',').map(v => v.split(':')) : []

      /** @type {Map<string, string>} */
      const newIdMap = new Map(idMap.filter(([v, id]) => v <= n && VIDEO_ID_REGEX.test(id)))

      if (idMap.length > newIdMap.size) {
        history.replaceState(null, null, `#/${layout}/` + mapToStr(newIdMap))
      }

      const videoIds = Object.assign({}, this.videoIds)

      for (const n of Object.keys(videoIds)) {
        if (!newIdMap.has(n)) {
          videoIds[n] = ''
          this.getRefs(`url-${n}`)?.classList.remove('hidden')
        }
      }

      // データの更新
      for (const [n, id] of newIdMap) {
        const int = Math.floor(n)
        videoIds[int] = id
        VIDEO_ID_MAP.set(int, id)
      }

      this.videoIds = videoIds

      // ハッシュ更新による再描画
      if (isHashChange) {
        this.$nextTick(() => {
          for (const [n, id] of newIdMap) {
            this.getRefs(`url-${n}`).classList.add('hidden')
            this.getRefs(`input-${n}`).value = `https://www.youtube.com/watch?v=${id}`
          }
        })
      }

      isHashChange = true
    }

    window.addEventListener('hashchange', onHashChange)
    onHashChange()

    // スクロールバー調整
    if (this.$el.offsetWidth === this.$el.clientWidth) {
      scrollElement = this.$el
    } else {
      scrollElement = new SimpleBar(this.$el).getScrollElement()
    }

    this.$el.classList.add('init')

    // フルスクリーンが有効ならばイベントを追加
    if (this.enabled) {
      this.$el.addEventListener('fullscreenchange', () => {
        this.isFullScreen = !!document.fullscreenElement
      })
    }
  }
}
</script>

<style lang="scss">
@import "./scss/main.scss";
</style>
