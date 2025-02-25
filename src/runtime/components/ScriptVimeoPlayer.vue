<script setup lang="ts">
/// <reference types="vimeo__player" />
import { type HTMLAttributes, type ImgHTMLAttributes, computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { defu } from 'defu'
import type { ElementScriptTrigger } from '../types'
import { useAsyncData, useElementScriptTrigger, useHead, useScriptVimeoPlayer } from '#imports'

const props = withDefaults(defineProps<{
  // custom
  trigger?: ElementScriptTrigger
  placeholderAttrs?: ImgHTMLAttributes
  rootAttrs?: HTMLAttributes
  aboveTheFold?: boolean
  // copied from @types/vimeo__player
  id?: number | undefined
  url?: string | undefined
  autopause?: boolean | undefined
  autoplay?: boolean | undefined
  background?: boolean | undefined
  byline?: boolean | undefined
  color?: string | undefined
  controls?: boolean | undefined
  dnt?: boolean | undefined
  height?: number | undefined
  // eslint-disable-next-line vue/prop-name-casing
  interactive_params?: string | undefined
  keyboard?: boolean | undefined
  loop?: boolean | undefined
  maxheight?: number | undefined
  maxwidth?: number | undefined
  muted?: boolean | undefined
  pip?: boolean | undefined
  playsinline?: boolean | undefined
  portrait?: boolean | undefined
  responsive?: boolean | undefined
  speed?: boolean | undefined
  quality?: Vimeo.VimeoVideoQuality | undefined
  texttrack?: string | undefined
  title?: boolean | undefined
  transparent?: boolean | undefined
  width?: number | undefined
}>(), {
  trigger: 'mousedown',
  width: 640,
  height: 480,
  loop: false,
  controls: true,
})

const emits = defineEmits<TEmits>()

type EventMap<E extends keyof Vimeo.EventMap> = [event: Vimeo.EventMap[E], player: Vimeo]

// eslint-disable-next-line ts/consistent-type-definitions
type TEmits = {
  play: EventMap<'play'>
  playing: EventMap<'playing'>
  pause: EventMap<'pause'>
  ended: EventMap<'ended'>
  timeupdate: EventMap<'timeupdate'>
  progress: EventMap<'progress'>
  seeking: EventMap<'seeking'>
  seeked: EventMap<'seeked'>
  texttrackchange: EventMap<'texttrackchange'>
  chapterchange: EventMap<'chapterchange'>
  cuechange: EventMap<'cuechange'>
  cuepoint: EventMap<'cuepoint'>
  volumechange: EventMap<'volumechange'>
  playbackratechange: EventMap<'playbackratechange'>
  bufferstart: EventMap<'bufferstart'>
  bufferend: EventMap<'bufferend'>
  error: EventMap<'error'>
  loaded: EventMap<'loaded'>
  durationchange: EventMap<'durationchange'>
  fullscreenchange: EventMap<'fullscreenchange'>
  qualitychange: EventMap<'qualitychange'>
  camerachange: EventMap<'camerachange'>
  resize: EventMap<'resize'>
  enterpictureinpicture: EventMap<'enterpictureinpicture'>
  leavepictureinpicture: EventMap<'leavepictureinpicture'>
}

const events: (keyof TEmits)[] = [
  'play',
  'playing',
  'pause',
  'ended',
  'timeupdate',
  'progress',
  'seeking',
  'seeked',
  'texttrackchange',
  'chapterchange',
  'cuechange',
  'cuepoint',
  'volumechange',
  'playbackratechange',
  'bufferstart',
  'bufferend',
  'error',
  'loaded',
  'durationchange',
  'fullscreenchange',
  'qualitychange',
  'camerachange',
  'resize',
  'enterpictureinpicture',
  'leavepictureinpicture',
]

const elVimeo = ref()
const rootEl = ref()

const trigger = useElementScriptTrigger({ trigger: props.trigger, el: rootEl })
let clickTriggered = false
if (props.trigger === 'mousedown') {
  trigger.then(() => {
    clickTriggered = true
  })
}
const ready = ref(false)
const { $script } = useScriptVimeoPlayer({
  scriptOptions: {
    trigger,
  },
})

if (import.meta.server) {
  // dns-prefetch https://i.vimeocdn.com
  useHead({
    link: [
      {
        rel: props.aboveTheFold ? 'preconnect' : 'dns-prefetch',
        href: 'https://i.vimeocdn.com',
      },
    ],
  })
}

const { data: payload } = useAsyncData(
  `vimeo-embed:${props.id}`,
  // TODO ideally we cache this
  () => $fetch(`https://vimeo.com/api/v2/video/${props.id}.json`).then(res => (res as any)[0]),
  {
    watch: [() => props.id],
  },
)

const placeholder = computed(() => payload.value?.thumbnail_large)

let player: Vimeo | undefined
// we can't directly expose the player as vue will break the proxy
defineExpose({
  play: () => player?.play(),
  pause: () => player?.pause(),
  getDuration: () => player?.getDuration(),
  getCurrentTime: () => player?.getCurrentTime(),
  setCurrentTime: (time: number) => player?.setCurrentTime(time),
  getVolume: () => player?.getVolume(),
  setVolume: (volume: number) => player?.setVolume(volume),
  getPaused: () => player?.getPaused(),
  getEnded: () => player?.getEnded(),
  getLoop: () => player?.getLoop(),
  setLoop: (loop: boolean) => player?.setLoop(loop),
  getPlaybackRate: () => player?.getPlaybackRate(),
  setPlaybackRate: (rate: number) => player?.setPlaybackRate(rate),
})

onMounted(() => {
  $script.then(async ({ Vimeo }) => {
    // filter props for false values
    player = new Vimeo.Player(elVimeo.value, props)
    if (clickTriggered) {
      player!.play()
      clickTriggered = false
    }
    for (const event of events) {
      player!.on(event, (e: EventMap<typeof event>) => {
        // @ts-expect-error ts isn't able to infer the correct event type
        emits(event, e, player)
        if (event === 'loaded')
          ready.value = true
      })
    }
  })

  watch(() => props.id, (v) => {
    v && player?.loadVideo(Number(v))
  })
})

const rootAttrs = computed(() => {
  return defu(props.rootAttrs, {
    'aria-busy': $script.status.value === 'loading',
    'aria-label': $script.status.value === 'awaitingLoad'
      ? 'Vimeo Player - Placeholder'
      : $script.status.value === 'loading'
        ? 'Vimeo Player - Loading'
        : 'Vimeo Player - Loaded',
    'aria-live': 'polite',
    'role': 'application',
    'style': {
      maxWidth: '100%',
      width: `${props.width}px`,
      height: `'auto'`,
      aspectRatio: `${props.width}/${props.height}`,
      position: 'relative',
      backgroundColor: 'black',
    },
  }) as HTMLAttributes
})

const placeholderAttrs = computed(() => {
  return defu(props.placeholderAttrs, {
    src: placeholder.value,
    alt: '',
    loading: props.aboveTheFold ? 'eager' : 'lazy',
    // @ts-expect-error untyped
    fetchpriority: props.aboveTheFold ? 'high' : undefined,
    style: {
      cursor: 'pointer',
      width: '100%',
      objectFit: 'contain',
      height: '100%',
    },
  } satisfies ImgHTMLAttributes)
})

onBeforeUnmount(() => player?.unload())
</script>

<template>
  <div ref="rootEl" v-bind="rootAttrs">
    <div v-show="ready" ref="elVimeo" class="vimeo-player" style="width: 100%; height: 100%; max-width: 100%;" />
    <slot v-if="!ready" v-bind="payload" :placeholder="placeholder" name="placeholder">
      <img v-if="placeholder" v-bind="placeholderAttrs">
    </slot>
    <slot v-if="$script.status.value === 'loading'" name="loading">
      <ScriptLoadingIndicator color="white" />
    </slot>
    <slot v-if="$script.status.value === 'awaitingLoad'" name="awaitingLoad" />
    <slot />
  </div>
</template>

<style>
.vimeo-player iframe {
  max-width: 100% !important;
}
</style>
