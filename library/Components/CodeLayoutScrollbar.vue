<template>
  <div class="code-layout-scrollbar" @wheel="mouseWheel">
    <div 
      ref="container" 
      :class="'scroll-content ' + scroll + ' ' + containerClass"
      @scroll="calcScrollBarPosition"
    >
      <slot />
    </div>
    <div 
      v-if="scrollBarX.show"
      ref="scrollBarRefX"
      class="scrollbar horizontal"
      :style="{ height: `${scrollBarSize}px` }"
    >
      <div
        class="thumb"
        :style="{ left: `${scrollBarX.pos}%`, width: `${scrollBarX.size}%` }"
        @mousedown="thumbDrageHandlerX"
      />
    </div>
    <div 
      v-if="scrollBarY.show"
      ref="scrollBarRefY"
      class="scrollbar vertical"
      :style="{ width: `${scrollBarSize}px` }"
    >
      <div 
        class="thumb"
        :style="{ top: `${scrollBarY.pos}%`, height: `${scrollBarY.size}%` }"
        @mousedown="thumbDrageHandlerY"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import HtmlUtils from '../Utils/HtmlUtils';
import { ref, onMounted, onBeforeUnmount, nextTick, type PropType, reactive } from "vue";
import { createMouseDragHandler } from '../Composeable/MouseHandler';
import { useResizeChecker } from "../Composeable/ResizeChecker";


const props = defineProps({	
  /**
   * Scroll direction
   */
  scroll: {
    type: String as PropType<'both'|'none'|'vertical'|'horizontal'>,
    default: 'both'
  },
  /**
   * Scroll bar size (pixel)
   */
  scrollBarSize: {
    type: Number,
    default: 8
  },
  /**
   * CSS class of inner container
   */
  containerClass: {
    type: String,
    default: ''
  },
})

const container = ref<HTMLElement>();
const scrollBarRefX = ref<HTMLElement>();
const scrollBarRefY = ref<HTMLElement>();
  
const scrollBarX = reactive({
  show: false,
  size: 0,
  sizeRaw: 0,
  pos: 0,
});
const scrollBarY = reactive({
  show: false,
  size: 0,
  sizeRaw: 0,
  pos: 0,
});

let lastCalcScrollScrollWidth = 0;
let lastCalcScrollScrollHeight = 0;
let lastCalcScrollWidth = 0;
let lastCalcScrollHeight = 0;

const observer = new MutationObserver(() => calcScroll());
 
// 配置观察选项
const config = { attributes: true, childList: true };

function calcScrollBarPosition() {
  if (!container.value) 
    return;
  if (scrollBarX.show) {
    const sizePrecent = (container.value.offsetWidth / container.value.scrollWidth);
    scrollBarX.sizeRaw = sizePrecent * container.value.offsetWidth;
    scrollBarX.size = sizePrecent * 100;
    scrollBarX.pos = (container.value.scrollLeft / (container.value.scrollWidth - container.value.offsetWidth)) * (100 - scrollBarX.size);
    if (sizePrecent >= 1)
      scrollBarX.show = false;
  }
  if (scrollBarY.show) {
    const sizePrecent = (container.value.offsetHeight / container.value.scrollHeight);
    scrollBarY.sizeRaw = sizePrecent * container.value.offsetHeight;
    scrollBarY.size = sizePrecent * 100;
    scrollBarY.pos = (container.value.scrollTop / (container.value.scrollHeight - container.value.offsetHeight)) * (100 - scrollBarY.size);
    if (sizePrecent >= 1)
      scrollBarY.show = false;
  }
}

function calcScroll(force = false) {
  
  if (!container.value) 
    return;
  let canScrollX = props.scroll === 'horizontal' || props.scroll === 'both';
  let canScrollY = props.scroll === 'vertical' || props.scroll === 'both';

  const xScrollValueChanged = canScrollX && (lastCalcScrollScrollWidth !== container.value.scrollWidth
    || lastCalcScrollWidth !== container.value.offsetWidth); 
  const yScrollValueChanged = canScrollY && (lastCalcScrollScrollHeight !== container.value.scrollHeight 
    || lastCalcScrollHeight !== container.value.offsetHeight); 

  if (!force && !xScrollValueChanged && !yScrollValueChanged)
    return;

  const style = window.getComputedStyle(container.value);

  if (style.overflow === 'hidden' || style.overflowX === 'hidden') 
    canScrollX = false;
  if (style.overflow === 'hidden' || style.overflowY === 'hidden') 
    canScrollY = false;

  scrollBarX.show = canScrollX;
  scrollBarY.show = canScrollY;

  calcScrollBarPosition();

  lastCalcScrollWidth = container.value.offsetWidth;
  lastCalcScrollHeight = container.value.offsetHeight
  lastCalcScrollScrollWidth = container.value.scrollWidth;
  lastCalcScrollScrollHeight = container.value.scrollHeight;
}

//只有横向滚动时，使鼠标可以直接滚动
function mouseWheel(e: WheelEvent) {
  if (props.scroll == 'horizontal') {
    if (e.deltaMode == 0) {
      container.value?.scrollTo({
        left: container.value.scrollLeft + (e.deltaY > 0 ? 30 : -30),
      });
    }
  }
}

//滚动条滚动处理
let mouseDragDownThumbX = 0, mouseDragDownX = 0;
let mouseDragDownThumbY = 0, mouseDragDownY = 0;
const thumbDrageHandlerX = createMouseDragHandler({
  onDown(e) { 
    mouseDragDownThumbX = e.offsetX;
    if (scrollBarRefX.value) 
      mouseDragDownX= HtmlUtils.getLeft(scrollBarRefX.value);
    return true;
  },
  onMove(downPos, movedPos, e) {
    if (container.value && scrollBarRefX.value) {
      container.value.scrollLeft = (e.x - mouseDragDownThumbX - mouseDragDownX) 
        / (container.value.offsetWidth - scrollBarX.sizeRaw)
        * (container.value.scrollWidth - container.value.offsetWidth);
      e.preventDefault();
      e.stopPropagation();
    }
  },
  onUp() {},
})
const thumbDrageHandlerY = createMouseDragHandler({
  onDown(e) { 
    mouseDragDownThumbY = e.offsetY;
    if (scrollBarRefY.value) 
      mouseDragDownY = HtmlUtils.getTop(scrollBarRefY.value);
    return true;
  },
  onMove(downPos, movedPos, e) {
    if (container.value && scrollBarRefY.value) {
      container.value.scrollTop = (e.y - mouseDragDownThumbY - mouseDragDownY) 
        / (container.value.offsetHeight - scrollBarY.sizeRaw)
        * (container.value.scrollHeight - container.value.offsetHeight);
      e.preventDefault();
      e.stopPropagation();
    }
  },
  onUp() {},
})

//大小更改事件
const {
  startResizeChecker,
  stopResizeChecker,
} = useResizeChecker(
  container, 
  () => calcScroll(), 
  () => calcScroll()
);

onMounted(() => {
  console.warn('CodeLayoutScrollbar in codelayout is deprecated and no longer maintain, please use @imengyu/vue-scroll-rect package instead.');
  nextTick(() => {
    setTimeout(() => calcScroll(true), 200);
    calcScroll(true);
    startResizeChecker();
    observer.observe(container.value!, config);
  });
});
onBeforeUnmount(() => {
  stopResizeChecker();
  observer.disconnect();
});

export interface CodeLayoutScrollbarInstance {
  /**
   * Force update scrollbar state
   */
  refreshScrollState(): void;
  /**
   * Get scroll containerElement
   */
  getScrollContainer(): HTMLElement | undefined;
  /**
   * Scroll to position
   * @param x X scroll pos (pixel)
   * @param y Y scroll pos (pixel)
   */
  scrollTo(x: number, y: number): void;
  /**
   * Scroll to top
   */
  scrollToTop() : void;
  /**
   * Scroll to bottom
   */
  scrollToBottom() : void;
}

defineExpose<CodeLayoutScrollbarInstance>({
  refreshScrollState() {
    calcScroll(true);
  },
  getScrollContainer() {
    return container.value;
  },
  scrollTo(x: number, y: number) {
    container.value?.scrollTo(x, y);
  },
  scrollToTop() {
    container.value?.scrollTo(0, 0);
  },
  scrollToBottom() {
    container.value?.scrollTo(container.value.scrollWidth, container.value.scrollHeight);
  },
})

</script>

<style lang="scss">
.code-layout-scrollbar {
  position: relative;
  width: 100%;
  height: 100%;
  margin: 0;

  > .scroll-content {
    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;

    &::-webkit-scrollbar {
      width: 0;
      height: 0;
    }
    &::-ms-scrollbar {
      width: 0;
      height: 0;
    }

    &.both {
      overflow: scroll;
    }
    &.horizontal {
      overflow-x: scroll;
    }
    &.vertical {
      overflow-y: scroll;
    }
  }

  &:hover > .scrollbar {
    opacity: 1;
    transition: opacity 0.1s;
  }

  > .scrollbar {
    position: absolute;
    opacity: 0;
    transition: opacity 1.5s;

    &:hover {
      opacity: 1;
    }
    .thumb {
      position: absolute;
      background-color: var(--code-layout-color-scrollbar-thumb);

      &:hover {
        background-color: var( --code-layout-color-scrollbar-thumb-light);
      }
    }

    &.horizontal {
      left: 0;
      bottom: 0;
      right: 0;

      .thumb {
        top: 0;
        bottom: 0;
      }
    }
    &.vertical {
      top: 0;
      bottom: 0;
      right: 0;

      .thumb {
        left: 0;
        right: 0;
      }
    }
  }
}
</style>