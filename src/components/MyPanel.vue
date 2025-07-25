<script setup lang="ts">
import { nextTick, ref, toRefs, onMounted, onUnmounted, watch } from 'vue'
import { NPopover, NMenu, MenuOption, NText, NButton, NIcon } from 'naive-ui'
import { CaretLeft, CaretRight } from '@vicons/fa'
// @ts-ignore
import getCaretCoordinates from 'textarea-caret'
import emojiRegex from 'emoji-regex'
import {
  process,
  changePage,
  selectCandidateOnCurrentPage
} from '../workerAPI'
import {
  text,
  autoCopy,
  forceVertical,
  loading,
  hideComment,
  changeLanguage,
  selectIME,
  syncOptions
} from '../control'
import { isMobile, getTextarea } from '../util'

const props = defineProps<{
  debugMode?: boolean
}>()

const {
  debugMode
} = toRefs(props)

const mouseX = ref<number>(0)
const mouseY = ref<number>(0)
const dragging = ref<boolean>(false)
const dragged = ref<boolean>(false)
const x = ref<number>(0)
const y = ref<number>(0)

const preEditHead = ref<string>('')
const preEditBody = ref<string>('')
const preEditTail = ref<string>('')

const menuOptions = ref<MenuOption[]>([])
const highlighted = ref<number>(0)

const prevDisabled = ref<boolean>(true)
const nextDisabled = ref<boolean>(false)

// Call to librime is async so there's a delay from editing=true to showMenu=true
const editing = ref<boolean>(false)
const showMenu = ref<boolean>(false)
const xOverflow = ref<boolean>(false)
const exclusiveShift = ref<boolean>(false)

async function debug (e: KeyboardEvent, rimeKey: string) {
  editing.value = true
  await input(rimeKey);
  (e.target as HTMLElement).focus()
}

const RIME_KEY_MAP: {[key: string]: string | undefined} = {
  Escape: 'Escape',
  F4: 'F4',
  Backspace: 'BackSpace',
  Delete: 'Delete',
  Tab: 'Tab',
  Enter: 'Return',
  Home: 'Home',
  End: 'End',
  PageUp: 'Page_Up',
  PageDown: 'Page_Down',
  Alt: 'Alt_L',
  ArrowUp: 'Up',
  ArrowRight: 'Right',
  ArrowDown: 'Down',
  ArrowLeft: 'Left',
  '~': 'asciitilde',
  '`': 'quoteleft',
  '!': 'exclam',
  '@': 'at',
  '#': 'numbersign',
  $: 'dollar',
  '%': 'percent',
  '^': 'asciicircum',
  '&': 'ampersand',
  '*': 'asterisk',
  '(': 'parenleft',
  ')': 'parenright',
  '-': 'minus',
  _: 'underscore',
  '+': 'plus',
  '=': 'equal',
  '{': 'braceleft',
  '[': 'bracketleft',
  '}': 'braceright',
  ']': 'bracketright',
  ':': 'colon',
  ';': 'semicolon',
  '"': 'quotedbl',
  "'": 'apostrophe',
  '|': 'bar',
  '\\': 'backslash',
  '<': 'less',
  ',': 'comma',
  '>': 'greater',
  '.': 'period',
  '?': 'question',
  '/': 'slash',
  ' ': 'space'
}

const CONTROL_ALLOWLIST = ['`']

function isPrintable (key: string) {
  return /^[a-z0-9!"#$%&'()*+,./:;<=>?@[\] ^_`{|}~\\-]$/i.test(key)
}

const regex = emojiRegex()
function isEmoji (c: string) {
  regex.lastIndex = 0
  return regex.test(c)
}

function insert (toInsert: string) {
  const textarea = getTextarea()
  const { selectionStart, selectionEnd } = textarea
  text.value = text.value.slice(0, selectionStart) + toInsert + text.value.slice(selectionEnd)
  if (autoCopy.value) {
    navigator.clipboard.writeText(text.value)
  }
  nextTick(() => {
    textarea.selectionEnd = selectionStart + toInsert.length
  })
}

async function analyze (result: RIME_RESULT, rimeKey: string) {
  const textarea = getTextarea()
  if (!('updatedSchema' in result) && result.updatedOptions) {
    syncOptions(result.updatedOptions)
  }
  if (result.state === 0) { // COMMITTED
    editing.value = false
    showMenu.value = false
    dragged.value = false
    insert(result.committed)
  } else if (result.state === 1) { // ACCEPTED
    preEditHead.value = result.head
    preEditBody.value = result.body
    preEditTail.value = result.tail
    highlighted.value = result.highlighted
    // @ts-ignore
    menuOptions.value = result.candidates.map((candidate, i) => {
      let label = `${result.selectLabels?.[i] || i + 1} ${candidate.text}`
      if (candidate.comment && (hideComment.value === false || (hideComment.value === 'emoji' && !isEmoji(candidate.text)))) {
        label += ' ' + candidate.comment
      }
      return { label, key: i }
    })
    prevDisabled.value = result.page === 0
    nextDisabled.value = result.isLastPage
    if (!showMenu.value) {
      showMenu.value = true
      xOverflow.value = false
    }
    nextTick(() => {
      const panelWidth = document.querySelector('.n-popover')!.getBoundingClientRect().width
      if (panelWidth > textarea.getBoundingClientRect().width) {
        xOverflow.value = true
      }
    })
    if (result.committed) {
      insert(result.committed)
    }
  } else { // REJECTED, UNHANDLED
    editing.value = false
    showMenu.value = false
    if (result.state === 2 && result.updatedSchema) {
      await selectIME(result.updatedSchema.split('/')[0])
    }
    if (result.state === 3 && isPrintable(rimeKey)) {
      insert(rimeKey)
    }
  }
  textarea.focus()
}

async function input (rimeKey: string) {
  const result = await process(rimeKey)
  return analyze(result, rimeKey)
}

// begin: code specific to Android Chromium
let androidChromium = false
let acStart = 0
let acEnd = 0

watch(text, (acNewText, acText) => {
  if (!androidChromium) {
    return
  }
  androidChromium = false
  if (acText.length + 1 === acNewText.length &&
      acText.substring(0, acStart) === acNewText.substring(0, acStart) &&
      acText.substring(acEnd) === acNewText.substring(acEnd + 1)) {
    const textarea = getTextarea()
    text.value = acText
    nextTick(() => {
      editing.value = true
      textarea.selectionEnd = acStart
      input(acNewText[acStart])
    })
  }
})
// end: code specific to Android Chromium

function onKeydown (e: KeyboardEvent) {
  if (debugMode.value || loading.value) {
    return
  }
  const { code, key } = e
  const textarea = getTextarea()
  // begin: code specific to Android Chromium
  if (key === 'Unidentified') {
    androidChromium = true
    acStart = textarea.selectionStart
    acEnd = textarea.selectionEnd
    return
  }
  // end: code specific to Android Chromium
  if (key === 'Shift') {
    exclusiveShift.value = true
    return
  }
  exclusiveShift.value = false

  const isPrintableKey = isPrintable(key)
  const isAlt = key === 'Alt'
  const hasControl = e.getModifierState('Control')
  const hasMeta = e.getModifierState('Meta')
  const hasAlt = e.getModifierState('Alt')
  const hasShift = e.getModifierState('Shift')
  const isShortcut = hasControl || hasMeta || hasAlt || (hasShift && !isPrintableKey)

  // In edit mode, rime handles every keydown;
  // In non-edit mode, only when the textarea is focused and a printable key is down (w/o modifier) will activate rime.
  if (!editing.value) {
    if (document.activeElement !== textarea || (!isPrintableKey && key !== 'F4')) {
      return
    }
    // Don't send Control+x, Meta+x, Alt+x to librime, but allow them with Shift
    if (isShortcut && !hasShift && !(hasControl && CONTROL_ALLOWLIST.includes(key))) {
      return
    }
  }
  let rimeKey: string | undefined
  const wrap = (s: string) => `{${s}}`
  if (isShortcut || !isPrintableKey) {
    rimeKey = /^[0-9a-z]$/i.test(key) ? key : RIME_KEY_MAP[key]
    if (rimeKey === undefined) {
      return
    }
    if (isAlt && code === 'AltRight') {
      rimeKey = 'Alt_R'
    }
    const modifiers: string[] = []
    if (hasControl) {
      modifiers.push('Control')
    }
    if (hasMeta) {
      modifiers.push('Meta')
    }
    if (hasAlt && !isAlt) {
      modifiers.push('Alt')
    }
    if (hasShift) {
      modifiers.push('Shift')
    }
    modifiers.push(rimeKey)
    rimeKey = wrap(modifiers.join('+'))
  } else if (code.startsWith('Numpad')) {
    rimeKey = wrap(`KP_${code.substring(6)}`)
  } else {
    rimeKey = key
  }

  if (!dragged.value) {
    const box = textarea.getBoundingClientRect()
    const caret: { top: number, left: number, height: number } = getCaretCoordinates(textarea, textarea.selectionStart)
    x.value = box.x + caret.left
    y.value = isMobile.value ? 8 : box.y + caret.top + caret.height - textarea.scrollTop
  }
  editing.value = true
  e.preventDefault()
  input(rimeKey)
}

function onKeyup (e: KeyboardEvent) {
  if (debugMode.value || loading.value) {
    return
  }
  const { key } = e
  if (key === 'Shift' && exclusiveShift.value) {
    changeLanguage()
  }
  exclusiveShift.value = false
  if (editing.value) {
    input(`{Release+${RIME_KEY_MAP[key] || key}}`)
  }
}

async function onClick (key: number) {
  const result = JSON.parse(await selectCandidateOnCurrentPage(key))
  return analyze(result, '')
}

async function onPageChange (backward: boolean) {
  const result = JSON.parse(await changePage(backward))
  return analyze(result, '')
}

function singleTouch (e: TouchEvent) {
  return e.touches.length === 1 ? e.touches[0] : undefined
}

function handleDown (clientX: number, clientY: number) {
  mouseX.value = clientX
  mouseY.value = clientY
  // As flip is turned on, update x to actual position to avoid layout shift on click
  const panel = document.querySelector('.n-popover')!
  x.value = panel.getBoundingClientRect().left
  dragging.value = true
}

function onMousedown (e: MouseEvent) {
  handleDown(e.clientX, e.clientY)
}

function onTouchstart (e: TouchEvent) {
  const touch = singleTouch(e)
  touch && handleDown(touch.clientX, touch.clientY)
}

function handleMove (clientX: number, clientY: number) {
  if (!dragging.value) {
    return
  }
  dragged.value = true
  x.value += clientX - mouseX.value
  y.value += clientY - mouseY.value
  mouseX.value = clientX
  mouseY.value = clientY
}

function onMousemove (e: MouseEvent) {
  handleMove(e.clientX, e.clientY)
}

function onTouchmove (e: TouchEvent) {
  const touch = singleTouch(e)
  touch && handleMove(touch.clientX, touch.clientY)
}

function onMouseupOrTouchend () {
  dragging.value = false
}

onMounted(() => {
  document.addEventListener('keydown', onKeydown)
  document.addEventListener('keyup', onKeyup)
  document.addEventListener('mousemove', onMousemove)
  document.addEventListener('touchmove', onTouchmove)
  document.addEventListener('mouseup', onMouseupOrTouchend)
  document.addEventListener('touchend', onMouseupOrTouchend)
})

onUnmounted(() => { // Cleanup for HMR
  document.removeEventListener('keydown', onKeydown)
  document.removeEventListener('keyup', onKeyup)
  document.removeEventListener('mousemove', onMousemove)
  document.removeEventListener('touchmove', onTouchmove)
  document.removeEventListener('mouseup', onMouseupOrTouchend)
  document.removeEventListener('touchend', onMouseupOrTouchend)
})

defineExpose({
  debug
})
</script>

<template>
  <n-popover
    :show="showMenu"
    :show-arrow="false"
    :x="x"
    :y="y"
    :flip="!dragging"
    placement="bottom-start"
    trigger="manual"
    style="cursor: move"
    @mousedown="onMousedown"
    @touchstart="onTouchstart"
  >
    <n-text type="success">
      {{ preEditHead }}
    </n-text>&nbsp;
    <n-text type="info">
      {{ preEditBody }}
    </n-text>&nbsp;
    {{ preEditTail }}
    <n-menu
      v-show="menuOptions.length"
      :options="menuOptions"
      :mode="forceVertical || isMobile || xOverflow ? 'vertical' : 'horizontal'"
      :value="highlighted"
      @update:value="onClick"
    />
    <n-button
      text
      :disabled="prevDisabled"
    >
      <n-icon
        :component="CaretLeft"
        @click="onPageChange(true)"
      />
    </n-button>
    <n-button
      text
      :disabled="nextDisabled"
    >
      <n-icon
        :component="CaretRight"
        @click="onPageChange(false)"
      />
    </n-button>
  </n-popover>
</template>

<style>
.n-menu-item-content-header {
  overflow: visible!important;
}
</style>
