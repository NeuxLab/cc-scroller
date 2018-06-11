<template>
  <div class='scroller select-list extension-list'>
    <draggable element="ol" class='scroller-wrapper list-group' :options="{ disabled: !sortable }" :list="items" ref='wrapper' :class="{'forbid-scroll':!scrollable , 'vertical': !horizontal, 'horizontal': horizontal }" @update="sortChange">
      <li class='scroller-item' v-for="(item, index) in items" :class="[ current === index ? active : '', landmarks[index] === 0 ? 'on-wrapper': '' ]" v-on="events" :key="item[key]" tabindex="-1" :data-index="index">
        <slot :index='index' :item="item"></slot>
      </li>
      <li ref="loading"><slot name="loading" v-if="showLoading==1"></slot><slot name="loadedAll" v-if="showLoading==2"></slot><slot name="empty" v-if="showLoading==3"></slot></li>
      </draggable>
  </div>
</template>

<script>
// 需要安装vueify
// https://github.com/electron-userland/electron-compile/issues/192
import { debounce } from 'lodash'
import draggable from 'vuedraggable'


export default {
  name: 'scroller',
  components: {
    draggable
  },
  data() {
    return {
      current: -1,
      offset: 0,
      wrapperLength: null,
      landmarks: [], // [-1, -1, 0, 0, 1, 1] marks
      scrollListener: null,
      loadingTimer: null,
      showLoading: 0
    }
  },
  props: {
    items: [],
    active: {
      type: String,
      default: "selected"
    },
    onFetch: Function,
    input: {
      type: Array,
      default: ["click", "contextmenu"]
    }
  },
  computed: {
    horizontal(){
      return this.$attrs["horizontal"] !== undefined
    },
    scrollable(){
      return this.$attrs["scrollable"] !== undefined
    },
    vertical(){
      return !this.horizontal;
    },
    sortable() {
      return this.$attrs["sortable"] !== undefined
    },
    performant() {
      return !this.paging;
    },
    paging() {
      return this.$attrs["paging"] !== undefined
    },
    key() {
      return this.$attrs["unique-key"]
    },
    focusable() {
      return this.$attrs["focusable"] !== undefined
    },
    events() {
      return this.input.reduce((last, v) => {
        last[v] = (event) => {
          this.current = parseInt(event.currentTarget.dataset.index);
        }
        return last;
      }, {});
    },
  },
  watch: {
    current(newValue) {
      let vm = this

      if (newValue < 0 || newValue >= vm.items.length) return
      var score = this.performant ? this.getScore(newValue) : this.landmarks[newValue]

      this.focusCurrent()

      if (score < 0) {
        vm.moveTo(newValue, true, () => { vm.$emit('current-change', newValue) })
      } else if (score > 0) {
        vm.moveTo(newValue, false, () => { vm.$emit('current-change', newValue) })
      } else {
        vm.$emit('current-change', newValue)
      }
    },
    items(newValue, oldValue) {
      if (!oldValue || !oldValue.length) {
        this.reset();
        return;
      }
      this.$nextTick(() => {
        this.focusCurrent();
      })
    },
    showLoading(newValue) {
      if (newValue > 0) {
        this.$nextTick(() => {
          this.$refs['loading'].scrollIntoView();
        })
      }
    }
  },
  methods: {
    reset() {
      this.$nextTick(this.initScrollable)
      this.current = -1
    },
    focusCurrent(){
      let target = this.$refs['wrapper'].$el.children[this.current]
      this.focusable && target && target.focus();
    },
    initScrollable() {
      let vm = this

      // marker wrapper length
      let wrapper = vm.$refs['wrapper'].$el;

      // reset scrollLeft
      wrapper.scrollTop = 0
      wrapper.scrollLeft = 0
      vm.offset = 0

      // calculate wrapperLength
      vm.wrapperLength = vm.vertical ? wrapper.offsetHeight : wrapper.offsetWidth
      console.log('wrapperLength', vm.wrapperLength)

      // calculate each children's position
      vm.landmark()
    },
    landmark(index) {
      // mark whether or not an item is in the wrapper view box
      // based on landmarks
      if (this.performant) {
        return index;
      }
      let vm = this
      let children = vm.$refs['wrapper'].$el.children

      vm.landmarks = []
      for (let index = 0; index < children.length; index++) {
        vm.landmarks[index] = this.getScore(index);
      }
      vm.$emit('landmarks-change', vm.landmarks)
      return vm.landmarks.indexOf(0);
    },
    moveTo(index, isFront = true, cb = null) {
      let vm = this
      let target = vm.$refs['wrapper'].$el.children[index]

      if (isFront) {
        // offset is target LEFT
        vm.offset = vm.vertical ? target.offsetTop : target.offsetLeft

        // apply offset to scroll
        vm.applyScroll(cb)
        // vm.$refs['wrapper'].scrollLeft = vm.offset
      } else {
        // offset + wrapperLength is target RIGHT
        vm.offset = (vm.vertical ? target.offsetTop + target.offsetHeight : target.offsetLeft + target.offsetWidth) - vm.wrapperLength

        // recalculate on-wrapper status
        // 右侧对齐标记哪些Item是需要显示后，再取第一个进行左侧对齐
        vm.moveTo(vm.landmark(index), true, cb)
      }
    },
    applyScroll(cb) {
      // this function only handle details on how to use requireAnimationFrame to scroll wrapper
      let vm = this
      let wrapper = vm.$refs['wrapper'].$el
      let startTime = null
      let startPosition = (vm.vertical ? wrapper.scrollTop : wrapper.scrollLeft)
      let endPosition = vm.offset
      let totalTime = 200 // 0.2 second for complete
      console.log('apply scroll', startPosition, endPosition)
      let scroll = function(timestamp) {
        if (!startTime) startTime = timestamp
        let progress = timestamp - startTime

        let currentOffset = startPosition + (endPosition - startPosition) * progress / totalTime

        // apply
        if (vm.vertical) {
          wrapper.scrollTop = currentOffset
        } else {
          wrapper.scrollLeft = currentOffset
        }

        if (progress < totalTime) {
          window.requestAnimationFrame(scroll)
        } else {
          if (typeof cb === 'function') cb()
        }
      }
      window.requestAnimationFrame(scroll)
    },
    setCurrent(index) {
      if (index < 0 || index >= this.items.length) return
      this.current = index
    },
    clearCurrent() {
      this.current = -1
    },
    nextPage() {
      let index = this.landmarks.indexOf(1)

      if (index >= 0) {
        this.moveTo(index)
      }
    },
    prevPage() {
      let index = this.landmarks.lastIndexOf(-1)
      if (index >= 0) {
        this.moveTo(index, false)
      }
    },
    nextCurrent(loop = false) {
      let target = this.current + 1
      if (!loop && target > this.items.length - 1) {
        return false;
      }
      this.current = target % this.items.length
      return true
    },
    prevCurrent(loop = false) {
      let target =this.current == -1 ? -1 : this.current - 1
      if (!loop && target < 0) return false;
      this.current = (target + this.items.length) % this.items.length
      return true;
    },
    sortChange(event) {
      this.$emit("sort-change", event)
    },
    getScore(index) {
      let children = this.$refs['wrapper'].$el.children
      let child = children[index]
      // if an item in the view, then (WRAPPER_LEFT - ITEM_LEFT) > 0 and (WRAPPER_RIGHT - ITEM_RIGHT) < 0
      // Outboundary happens when either too LEFT or too RIGHT, the multiple would always be greater than zero
      // thus, we use (WRAPPER_LEFT - ITEM_LEFT) * (WRAPPER_RIGHT * ITEM_RIGHT) > 0 to judge whether or not an item is in the wrapper current now
      let flag1 = this.offset - (this.vertical ? child.offsetTop : child.offsetLeft)
      let flag2 = (this.offset + this.wrapperLength) - (this.vertical ? child.offsetTop + child.offsetHeight : child.offsetLeft + child.offsetWidth)

      // vm.landmarks[index] = flag1 * flag2 <= 0
      if (flag1 > 0 && flag2 > 0) {
        return -1;
      } else if (flag1 < 0 && flag2 < 0) {
        return 1;
      } else {
        return 0;
      }
    },
    fetchData(before) {
      clearTimeout(this.loadingTimer);
      if (this.onFetch) {
        this.loadingTimer = setTimeout(() => {
          before && before();
          this.showLoading = 1;
        }, 200)
        this.onFetch(this.finishLoading)
      }
    },
    finishLoading(end, length) {
      clearTimeout(this.loadingTimer);
      if (end) {
        if (length) {
          this.showLoading = 2
        } else {
          this.showLoading = 3
        }
      } else {
        this.showLoading = 0;
      }
    }
  },
  mounted() {
    // add a listener on wrapper scroll, to update correct offset for landmark use
    let wrapper = this.$refs['wrapper'].$el;

    this.scrollListener = debounce(() => {
      this.offset = (this.vertical ? wrapper.scrollTop : wrapper.scrollLeft)
      this.landmark()
      if (this.items.length && this.getScore(this.items.length - 1) == 0) {
        this.fetchData();
      }
    }, 100)
    wrapper.addEventListener('scroll', this.scrollListener)
  }
}
</script>

<style lang="less">
.scroller {
  display: block;
  position: relative;
  height: 100%;
  width: 100%;

  ol.list-group {
    max-height: none;
    margin: 0;
  }

  .scroller-wrapper {
    width: 100%;
    height: 100%;
    position: relative;
    display: flex;
    overflow: scroll;

    &::-webkit-scrollbar {
      display: none;
    }

    &.horizontal {
      .scroller-item {
        margin: 0 0.2em;

        &:first-of-type {
          margin-left: 0;
        }
        &:last-of-type {
          margin-right: 0;
        }
      }
    }

    &.vertical {
      flex-direction: column;
    }

    &.forbid-scroll {
      overflow: hidden;
    }
  }
}
</style>
