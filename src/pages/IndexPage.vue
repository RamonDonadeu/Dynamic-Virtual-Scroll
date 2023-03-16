<template>
  <q-page class="flex flex-center" style="width: 40rem">
    <div style="height:500px; width: 40rem">
      <DynamicVirtualScroll :list="list" :extraLoadedItems="0">
        <template v-slot:itemList="{ itemList }">
          <div>{{ itemList.label }}</div>
        </template>
      </DynamicVirtualScroll>
    </div>
  </q-page>
</template>

<script>
import DynamicVirtualScroll from 'src/components/DynamicVirtualScroll.vue'
import { defineComponent } from 'vue'

export default defineComponent({
  name: 'IndexPage',
  components: { DynamicVirtualScroll },
  data () {
    return {
      list: []
    }
  },
  mounted () {
    for (let i = 0; i < 10000; i++) {
      this.list.push({ name: 'Test ' + i, label: 'Test ' + i })
    }
  }
})
</script>
