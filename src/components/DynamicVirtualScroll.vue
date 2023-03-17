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
          @click="bigger(item.virtualListIndex)"
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
let currentlyLoadingItemsToList = false

/** TODO
 * Check if list has 'virtualListIndex' and thorw an error
 * Currently only one list at the same time can be loaded
 * WHEN USING THE SCROLLBAR, IT GOES FROM THE INDEX WHERE IT WAS TO THE CURRENTLY INDEX, SECTION BY SECTION, IT DOSN'T JUMP
 * SOMETIMES THE LIST DISSAPEARS BECAUSE FOR SOME REASON, WHEN CHANGING THE SECTION, THE LIST GETS THE translateY VALUE WRONG SO THE LIST IS RENDERED AT A DIFFERENT HEIGHT FROM WHERE IT HAS TO BE
 */

const bigger = (index) => {
  indexRefs.value[index].style.height = '40px'
}

watch(props.list, () => {
  const itemsToInsert = []
  for (let i = 0; i < nItemsPerSection; i++) {
    itemsToInsert[i] = { ...props.list[i], virtualListIndex: i }
  }
  itemsLoaded.value = [...itemsToInsert]
  getSectionHeights(0)
  currentSection -= 1
  nextSection()
})

const getSectionHeights = (sectionToLoad) => {
  try {
    if (indexRefs.value.length === 0) return
    let heightSum = 0
    if (sectionToLoad !== 0) {
      heightSum = sectionHeights[sectionToLoad - 1]
    }
    console.log(`Insert ${sectionToLoad * 50} to ${sectionToLoad * 50 + 50}`)
    for (let i = 0; i < nItemsPerSection; i++) {
      const sectionMask = sectionToLoad * 50
      const height = parseInt(
        getComputedStyle(indexRefs.value[i + sectionMask]).height
      )
      if (!isNaN(height)) {
        heights.value[i + sectionMask] = height
        heightSum += height
      } else {
        debugger
      }
    }
    sectionHeights[sectionToLoad] = heightSum
    const virtualContainer = document.getElementsByClassName(
      'js-dynamicVirtualScroll__container'
    )[0]
    virtualContainer.style.height = heightSum + 'px'
  } catch (error) {
    console.log(error)
  }
}

const nextSection = (firstItemIndex) => {
  currentSection += 1
  let sumOfSectionHeight =
    currentSection === 0 ? 0 : sectionHeights[currentSection - 1]

  for (
    let i = currentSection * 50, j = 0;
    i < heights.value.length && j < nItemsPerSection;
    i++, j++
  ) {
    sumOfSectionHeight += heights.value[i]
    sumHeights[j] = sumOfSectionHeight
  }
  translateY = sumHeights[(firstItemIndex - 1) % nItemsPerSection]
}
const prevSection = (firstItemIndex) => {
  currentSection -= 1
  let sumOfSectionHeight =
    currentSection === 0 ? 0 : sectionHeights[currentSection - 1]

  for (
    let i = currentSection * 50, j = 0;
    i < heights.value.length && j < nItemsPerSection;
    i++, j++
  ) {
    sumOfSectionHeight += heights.value[i]
    sumHeights[j] = sumOfSectionHeight
  }

  translateY = sumHeights[(firstItemIndex - 1) % nItemsPerSection]
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
    if (j >= nItemsPerSection) {
      auxSumHeight += heights.value[i]
      item = auxSumHeight
    } else {
      item = sumHeights[j]
    }
    if (item < scrollTop) {
      firstVisibleItem = i
    } else if (item >= offsetHeight + scrollTop) {
      lastVisibleItem = i
      break
    }
  }
  // If lastVisibleItem ends up being -1 it means it has reached the end
  if (lastVisibleItem === -1) lastVisibleItem = sumHeights.length - 1

  const list = document.getElementsByClassName(
    'js-dynamicVirtualScroll__scroller'
  )[0]

  list.innerHtml = ''
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
  console.log(`Load from ${firstItemIndex} to ${lastItemIndex}`)
  if (firstItemIndex > lastItemIndex) return
  if (currentlyLoadingItemsToList) return
  itemsLoaded.value = []

  for (
    let i = firstItemIndex;
    i <= lastItemIndex && i < sectionHeights.length * 50;
    i++
  ) {
    itemsLoaded.value.push({ ...props.list[i], virtualListIndex: i })
  }
  console.log(
    `Old Height: ${translateY} | New Height: ${
      sumHeights[(firstItemIndex - 1) % nItemsPerSection]
    } `
  )
  firstItemIndex - 1 >= 0
    ? (translateY = sumHeights[(firstItemIndex - 1) % nItemsPerSection])
    : (translateY = 0)

  if (
    lastItemIndex >= heights.value.length &&
    lastItemIndex / nItemsPerSection >= sectionHeights.length
  ) {
    currentlyLoadingItemsToList = true
    loadNextSection()
  }
  if (firstItemIndex >= (currentSection + 1) * nItemsPerSection) {
    nextSection(firstItemIndex)
  } else if (
    firstItemIndex <= currentSection * nItemsPerSection &&
    currentSection > 0
  ) {
    prevSection(firstItemIndex)
  }
}

const loadNextSection = () => {
  console.log('Load Next section')
  const sectionToLoad = currentSection + 1
  const itemsToInsert = []
  for (let i = 0; i < nItemsPerSection; i++) {
    itemsToInsert[i] = {
      ...props.list[i + sectionToLoad * nItemsPerSection],
      virtualListIndex: i + sectionToLoad * nItemsPerSection
    }
    itemsLoaded.value.push(itemsToInsert[i])
    // console.log('Pushed from loadNextSection: ' + itemsToInsert[i].virtualListIndex)
  }
  setTimeout(() => {
    getSectionHeights(sectionToLoad)
    currentlyLoadingItemsToList = false
  }, 50)
}
onMounted(() => {
  const scroller = document.getElementsByClassName(
    'js-dynamicVirtualScroll'
  )[0]
  scroller.addEventListener(
    'scroll',
    function () {
      throtle(updateItemsList, 20)
    },
    false
  )
  window.addEventListener('keydown', handleKeyDown)
})

const setIndexRefs = (el) => {
  if (el) {
    if (indexRefs.value[el.id] !== null) {
      indexRefs.value[el.id] = el
    }
  }
}

let translateY = 0
const spacerStyle = () => {
  return {
    transform: 'translateY(' + translateY + 'px)'
  }
}

/** Handle Scroll with buttons */
const handleKeyDown = (event) => {
  console.log(event.keyCode)
  switch (event.keyCode) {
    // In case of up arrow key, move to the last item
    case 38:
      scrollDown()
      break

    // In case of down arrow key, move to the next item
    case 40:
      scrollUp()
      break
  }
}

const getFirstVisibleItem = () => {
  const scroller = document.getElementsByClassName(
    'js-dynamicVirtualScroll'
  )[0]
  // scrollTop -> Distance from visible scroll to the top
  // offsetHeight -> Space visible
  // scrollheight -> Height of all the scrollable list
  // eslint-disable-next-line no-unused-vars
  const { scrollTop, offsetHeight, scrollHeight } = scroller
  let firstVisibleItem = currentSection * 50
  let auxSumHeight = sectionHeights[currentSection]
  for (let i = currentSection * 50, j = 0; i < props.list.length; i++, j++) {
    let item
    if (j >= nItemsPerSection) {
      auxSumHeight += heights.value[i]
      item = auxSumHeight
    } else {
      item = sumHeights[j]
    }
    if (item > scrollTop) {
      firstVisibleItem = i
      break
    }
  }
  return firstVisibleItem
}

const goToItemWithIndex = (index) => {
  const indexSection = Math.floor(index / nItemsPerSection)
  let heightOfIndex = indexSection === 0 ? 0 : sectionHeights[indexSection - 1]
  for (let i = indexSection * 50; i < index; i++) {
    console.log(heightOfIndex)
    heightOfIndex += heights.value[i]
  }

  const scroller = document.getElementsByClassName(
    'js-dynamicVirtualScroll'
  )[0]
  scroller.scrollTop = heightOfIndex
}

const scrollUp = () => {
  const firstVisibleItem = getFirstVisibleItem()
  goToItemWithIndex(firstVisibleItem + 1)
}
const scrollDown = () => {
  const firstVisibleItem = getFirstVisibleItem()
  goToItemWithIndex(firstVisibleItem - 1)
}

// initialize throttlePause variable outside throttle function
let throttlePause
const throtle = (callback, time) => {
  // don't run the function if throttlePause is true
  if (throttlePause) return
  // If is calculating height of new section, return
  if (currentlyLoadingItemsToList) return

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
