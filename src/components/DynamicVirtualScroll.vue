<template lang="">
  <div class="js-dynamicVirtualScroll">
    <div
      class="js-dynamicVirtualScroll__container"
      ref="js-dynamicVirtualScroll__container"
    >
      <div
        class="js-dynamicVirtualScroll__scroller"
        ref="js-dynamicVirtualScroll__scroller"
        :style="spacerStyle()"
      >
        <div
          class="dynamicVirtualScroll__itemList"
          v-for="item in itemsLoaded"
          :key="item.virtualListIndex"
          :id="item.virtualListIndex"
          :ref="setIndexRefs"
        >
          <slot name="itemList" :itemList="item"></slot>
        </div>
      </div>
    </div>
  </div>
</template>
<script setup>
import { onMounted, ref, watch } from 'vue'

const props = defineProps({ list: Array, extraLoadedItems: Number })

const indexRefs = ref([])
const itemsLoaded = ref([])
const heights = ref([])
const sumHeights = []
const nItemsPerSection = 50
let currentSection = 0
const sectionHeights = []

/** TODO
 * Check if list has 'virtualListIndex' and thorw an error
 * Currently only one list at the same time can be loaded
 */

watch(props.list, () => {
  const itemsToInsert = []
  for (let i = 0; i < nItemsPerSection; i++) {
    itemsToInsert[i] = { ...props.list[i], virtualListIndex: i }
  }
  itemsLoaded.value = [...itemsToInsert]
  getSectionHeights(itemsToInsert, 0)
  currentSection -= 1
  nextSection()
})

const getSectionHeights = (itemsToInsert, sectionToLoad) => {
  if (indexRefs.value.length === 0) return
  let heightSum = 0
  if (sectionToLoad !== 0) {
    heightSum = sectionHeights[sectionToLoad - 1]
  }
  for (let i = 0; i < nItemsPerSection; i++) {
    const sectionMask = sectionToLoad * 50
    console.log(indexRefs.value[i + sectionMask])
    const height = parseInt(
      getComputedStyle(indexRefs.value[i + sectionMask]).height
    )
    heights.value[i + sectionMask] = height
    heightSum += height
  }
  sectionHeights[sectionToLoad] = heightSum
  const virtualContainer = document.getElementsByClassName(
    'js-dynamicVirtualScroll__container'
  )[0]
  virtualContainer.style.height = heightSum + 'px'
}

const nextSection = () => {
  currentSection += 1
  let sumOfSectionHeight = (currentSection === 0 ? 0 : sectionHeights[currentSection - 1])

  for (let i = currentSection * 50, j = 0; i < heights.value.length; i++, j++) {
    sumOfSectionHeight += heights.value[i]
    sumHeights[j] = sumOfSectionHeight
  }
}
const prevSection = () => {
  currentSection -= 1
  let sumOfSectionHeight = (currentSection === 0 ? 0 : sectionHeights[currentSection - 1])

  for (let i = currentSection * 50, j = 0; i < heights.value.length; i++, j++) {
    sumOfSectionHeight += heights.value[i]
    sumHeights[j] = sumOfSectionHeight
  }
}

const updateItemsList = () => {
  const scroller = document.getElementsByClassName(
    'js-dynamicVirtualScroll'
  )[0]
  // scrollTop -> Distance from visible scroll to the top
  // offsetHeight -> Space visible
  // scrollheight -> Height of all the scrollable list
  // eslint-disable-next-line no-unused-vars
  const { scrollTop, offsetHeight, scrollHeight } = scroller
  let firstVisibleItem = currentSection * 50
  let lastVisibleItem = -1
  let auxSumHeight = sectionHeights[currentSection]
  for (let i = currentSection * 50, j = 0; i < props.list.length; i++, j++) {
    let item
    if (j > sumHeights.length) {
      auxSumHeight += heights.value[i]
      item = auxSumHeight
    } else {
      item = sumHeights[j]
    }
    if (item < scrollTop) {
      firstVisibleItem = i + 1 + currentSection * 50
    } else if (item > offsetHeight + scrollTop) {
      lastVisibleItem = i
      break
    }
  }
  // If lastVisibleItem ends up being -1 it means it has reached the end
  if (lastVisibleItem === -1) lastVisibleItem = sumHeights.length - 1
  loadItemsToList(
    firstVisibleItem - props.extraLoadedItems >= 0
      ? firstVisibleItem - props.extraLoadedItems
      : 0,
    lastVisibleItem + props.extraLoadedItems <= props.list.length - 1
      ? lastVisibleItem + props.extraLoadedItems
      : props.list.length - 1
  )
}

const loadItemsToList = (firstItemIndex, lastItemIndex) => {
  console.log('Load from: ' + firstItemIndex + ' to ' + lastItemIndex)
  itemsLoaded.value = []
  for (let i = firstItemIndex; i <= lastItemIndex; i++) {
    itemsLoaded.value.push({ ...props.list[i], virtualListIndex: i })
  }
  firstItemIndex - 1 >= 0
    ? (translateY = sumHeights[firstItemIndex - 1])
    : (translateY = 0)
  if (lastItemIndex >= (currentSection + 1) * nItemsPerSection - 1 && currentSection + 1 >= sectionHeights.length) {
    loadNextSection()
  }
  if (firstItemIndex >= (currentSection + 1) * nItemsPerSection) {
    nextSection()
  } else if (firstItemIndex < (currentSection) * nItemsPerSection) {
    prevSection()
  }
}

const loadNextSection = () => {
  console.log('Load next section')
  const sectionToLoad = currentSection + 1
  const itemsToInsert = []
  for (let i = 0; i < nItemsPerSection; i++) {
    itemsToInsert[i] = {
      ...props.list[i + sectionToLoad * nItemsPerSection],
      virtualListIndex: i + sectionToLoad * nItemsPerSection
    }
    itemsLoaded.value.push(itemsToInsert[i])
  }
  setTimeout(() => {
    getSectionHeights(itemsToInsert, sectionToLoad)
  }, 10)
}
onMounted(() => {
  const scroller = document.getElementsByClassName(
    'js-dynamicVirtualScroll'
  )[0]
  scroller.addEventListener(
    'scroll',
    function () {
      throtle(updateItemsList, 25)
    },
    false
  )
})

const setIndexRefs = (el) => {
  if (el) {
    indexRefs.value[el.id] = el
  }
}

let translateY = 0
const spacerStyle = () => {
  return {
    transform: 'translateY(' + translateY + 'px)'
  }
}

// initialize throttlePause variable outside throttle function
let throttlePause
const throtle = (callback, time) => {
  // don't run the function if throttlePause is true
  if (throttlePause) return
  // set throttlePause to true after the if condition. This allows the function to be run once
  throttlePause = true

  // setTimeout runs the callback within the specified time
  setTimeout(() => {
    callback()

    // throttlePause is set to false once the function has been called, allowing the throttle function to loop
    throttlePause = false
  }, time)
}
</script>
<style scoped>
.js-dynamicVirtualScroll {
  overflow: auto;
  height: 100%;
}
.js-dynamicVirtualScroll__container {
  width: 100%;
  overflow: hidden;
  position: relative;
  will-change: auto;
}
</style>
