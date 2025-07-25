<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { NSpace, NCheckboxGroup, NCheckbox } from 'naive-ui'
import { LazyCache } from '@libreservice/lazy-cache'
import { getLanguage } from '../locale'
import fonts from '../../fonts.json'

const UbuntuFontMap = {
  'zh-CN': 'Noto Sans CJK SC',
  'zh-TW': 'Noto Sans CJK TC',
  'zh-HK': 'Noto Sans CJK HK',
  'zh-SG': 'Noto Sans CJK SC'
}

const WindowsFontMap = {
  'zh-CN': 'Microsoft YaHei',
  'zh-TW': 'Microsoft JhengHei',
  'zh-HK': 'Microsoft JhengHei',
  'zh-SG': 'Microsoft YaHei'
}

const macOSFontMap = {
  'zh-CN': 'PingFang SC',
  'zh-TW': 'PingFang TC',
  'zh-HK': 'PingFang HK',
  'zh-SG': 'PingFang SC'
}

const fontMap: { [key: string]: typeof fonts[0] } = {}
for (const font of fonts) {
  fontMap[font.fontFamily] = font
}

const language = getLanguage()
let defaultFont = ''
const UbuntuFont = UbuntuFontMap[language]
const WindowsFont = WindowsFontMap[language]
const macOSFont = macOSFontMap[language]

const storageKey = 'selectedFonts'

const lazyCache = new LazyCache('font')

async function loadFont (fontFamily: string) {
  if (loadedFonts.includes(fontFamily)) {
    return
  }
  loadedFonts.push(fontFamily)

  const {
    files,
    version
  } = fontMap[fontFamily]
  const fontFaces: string[] = []
  for (let i = 0; i < files.length; ++i) {
    const file = files[i]
     // @ts-ignore
    const url = (
       // @ts-ignore
      '__LIBRESERVICE_CDN__' // eslint-disable-line no-constant-condition
        ? `https://cdn.jsdelivr.net/npm/@libreservice/font-collection@${version}/dist/`
        : './'
    ) + file
    const buffer = await lazyCache.get(file, version, url)
    const blob = new Blob([buffer], { type: 'font/woff2' })
    const blobURL = URL.createObjectURL(blob)
    fontFaces.push(`
@font-face {
  font-family: "${fontFamily} ${i}";
  src: url(${blobURL}) format("woff2")
}`)
  }
  const style = document.createElement('style')
  style.innerHTML = fontFaces.join('\n')
  document.body.appendChild(style)
}

const loadedFonts: string[] = []
const selectedFonts = ref<string[]>([])

async function updateFonts (value: (string | number)[]) {
  selectedFonts.value = value as string[]
  localStorage.setItem(storageKey, JSON.stringify(selectedFonts.value))
  await Promise.all(selectedFonts.value.map(loadFont))
  const expandedFonts = selectedFonts.value.flatMap(fontFamily => Array.from({ length: fontMap[fontFamily].files.length }, (_, i) => `"${fontFamily} ${i}"`))
  document.body.style.fontFamily = [
    defaultFont,
    UbuntuFont,
    WindowsFont,
    macOSFont,
    ...expandedFonts
  ].join(', ')
}

try {
  const supportedFonts = fonts.map(({ fontFamily }) => fontFamily)
  updateFonts((JSON.parse(localStorage.getItem(storageKey) || '[]') as string[]).filter(font => supportedFonts.includes(font)))
} catch {}

onMounted(() => {
  defaultFont = getComputedStyle(document.body).fontFamily
})
</script>

<template>
  <n-space>
    Font for uncommon characters
    <n-checkbox-group
      :value="selectedFonts"
      @update:value="updateFonts"
    >
      <n-checkbox
        v-for="font of fonts"
        :key="font.name"
        :label="font.name"
        :value="font.fontFamily"
      />
    </n-checkbox-group>
  </n-space>
</template>
