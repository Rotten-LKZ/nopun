<script setup lang="ts">
import { ref, onMounted, computed, watch } from 'vue'
import { useStorage } from '@vueuse/core'
import { diffChars } from 'diff'
import type { Change } from 'diff'

const scrollEl = ref()
const diff = ref<Change[]>([])
const text = useStorage('text', '')
const showTextInput = useStorage('showTextInput', true)
const slashInput = useStorage('slashInput', '')
const textWithSlash = computed(() => text.value.replace(/(，|。|？|！|：|；|“|”|‘|’|、|（|）|——|…|《|》|\n|\t|　)+/g, '/').replace(/(^\/)|(\/$)/, ''))
const textWithoutPunctuation = computed(() => textWithSlash.value.replace(/\//g, ''))

onMounted(() => {
  const popup = document.getElementById('popup')
  const popupIframe: HTMLIFrameElement | null = document.querySelector('#popup iframe')
  scrollEl.value.addEventListener('mousewheel', wheelHandler, false)
  scrollEl.value.addEventListener('mouseup', mouseupHandler, false)

  function wheelHandler(event: any) {
    const detail = event.wheelDelta || event.detail
    const moveForwardStep = 1
    const moveBackStep = -1
    let step = 0
    if (detail < 0)
      step = moveForwardStep * 100
    else
      step = moveBackStep * 100
    
    if (document.body.scrollHeight < window.innerHeight)
      scrollEl.value.scrollLeft += -step
  }

  function mouseupHandler() {
    const txt = window.getSelection()?.toString()

    if (txt?.length === 1) {
      if (popup) popup.style.display = 'block'
      if (popupIframe) popupIframe.src = `https://dict.rotcool.me/${txt}`
    }
  }

  document.onmousedown = function(event: any) {
    if (event.target.id !== 'popup')
      if (popup) popup.style.display = 'none'
  }
})

watch(textWithoutPunctuation, refresh)

function check() {
  const diffRes = diffChars(textWithSlash.value, slashInput.value)
  const diffFilterRes: Array<Change> = []
    
  diffRes.forEach((value) => {
    if (value.added === undefined && value.removed === undefined) {
      const parts = value.value.split('/')
      for (let i = 0;i < parts.length - 1;i++) {
        diffFilterRes.push({ count: parts[i].length, value: parts[i] }, { count: 1, value: '/' })
      }
      diffFilterRes.push({ count: parts[parts.length - 1].length, value: parts[parts.length - 1] })
    } else {
      diffFilterRes.push(value)
    }
  })
  console.log(diffFilterRes)
  diff.value = diffFilterRes
}

function refresh(val: string) {
  slashInput.value = val
}
</script>

<template>
  <div id="popup">
    <iframe src="https://dict.rotcool.me/%E5%88%99" frameborder="0"></iframe>
  </div>
  <div class="display" ref="scrollEl">
    {{ textWithoutPunctuation }}
  </div>
  <div class="edit">
    <n-input
      v-model:value="text"
      type="textarea"
      placeholder="输入原文"
      v-show="showTextInput"
      style="height: 180px"
    />
    
    <div class="test">
      <div class="operate">
        <n-input
          v-model:value="slashInput"
          type="textarea"
          placeholder="标点"
          style="height: 240px"
        />
        <div class="btns">
          <n-button type="info" @click="check">
            检查
          </n-button>
          <n-button @click="refresh(textWithoutPunctuation)">
            重置
          </n-button>
          <n-button @click="diff = []">
            重置检查结果
          </n-button>
          <n-tooltip>
            <template #trigger>
              <n-button tag="div" disabled ghost>
                ?
              </n-button>
            </template>
            在觉得要断的地方加上“/”，点“检查”检查正误
          </n-tooltip>
          <n-switch :round="false" v-model:value="showTextInput">
            <template #checked>
              隐藏原文输入框
            </template>
            <template #unchecked>
              显示原文输入框
            </template>
          </n-switch>
        </div>
        
      </div>
      <span>
        紫色标注为多的部分，红色标注为少了的部分，绿色标注为正确的部分<br/>
        <span 
          v-for="(item, key) in diff"
          :key="key"
          :style="{
            color: item.added ? 'purple' : (item.removed ? 'red' : (item.value === '/' ? 'green' : 'grey')),
            fontWeight: item.added ? 'bold' : (item.removed ? 'bold' : (item.value === '/' ? 'bold' : 'normal')),
          }"
        >
          {{ item.value }}
        </span>
      </span>
    </div>
  </div>
</template>

<style lang="scss">
#popup {
  display: none;
  position: fixed;
  z-index: 99;
  overflow: hidden;
  background-color: var(--book-bg-color);
  bottom: 0;
  right: 0;

  iframe {
    width: 320px;
    height: 240px;
  }
}

body {
  --book-bg-color: #dadad0;

  margin: 0;
  padding: 0;
}

.display {
  width: 100%;
  height: 40vh;
  padding: 6px;
  box-sizing: border-box;
  line-height: 1.2;
  font-size: 1.2rem;
  overflow-x: auto;
  writing-mode: vertical-rl;
  background-color: var(--book-bg-color);
  font-family: "Noto Serif SC", STZhongsong, Roboto, serif;
}

.edit {
  margin: auto;
  padding: 6px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-top: 24px;
  max-width: 1000px;

  .test {
    display: flex;
    gap: 8px;
    
    .operate {
      flex: 3;
      gap: 8px;
      display: flex;
      flex-direction: column;
      
      .btns {
        display: flex;
        flex-wrap: wrap;
        align-items: center;
        gap: 8px;
      }
    }

    span {
      flex: 2;
    }
  }
}
@media (max-width: 781px) {
  .edit {
    .test {
      flex-direction: column;
    }
  }
}
</style>
