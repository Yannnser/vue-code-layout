<template>
  <div 
    v-if="panel.visible"
    ref="element"
    :class="[
      'code-layout-panel',
      open ? 'open' : 'closed',
      selected ? 'selected' : '',
      dragResizeable ? 'resizeable' : '',
      resizeDraggingSelf ? 'resizing-self' : '',
      resizeDragging ? 'resizing' : '',
      horizontal ? 'horizontal' : '',
      dragLightBoxState ? 'drag-enter' : '',
      dragSelfState ? 'dragging-self' : '',
      dragPanelState ? 'dragging' : '',
      opening ? 'opening' : '',
      `drag-over-${dragOverState}`,
    ]"
    :style="{
      width: horizontal && panelHeight ? `${panelHeight}px` : '100%',
      height: !horizontal && panelHeight ? `${panelHeight}px` : '100%',
    }"
    tabindex="0"
    @dragover="handleDragOver"
    @dragleave="handleDragLeave"
    @dragenter="handleDragEnter"
    @drop="handleDrop"
  > 
    <div 
      v-if="dragResizeable"
      ref="resizeDragger"
      :draggable="false"
      :class="[
        'drag-line',
        resizeDraggingSelf ? 'active' : ''
      ]" 
      @mousedown="resizeDragHandler"
    />
    <CodeLayoutCollapseTitle 
      v-if="!alone" 
      :draggable="panel.draggable && !resizeDragging"
      :showIconSmall="horizontal && !open"
      :iconSmall="panel.iconSmall"
      :title="panel.title"
      :tooltip="panel.tooltip"
      :actions="panel.actions"
      @click="handleHeaderClick"
      @contextmenu="onContextMenu(props.panel, $event)"
      @dragstart="handleDragStart(panel, $event)"
      @dragend="handleDragEnd"
    />
    <div 
      v-if="open"
      class="content"
      :draggable="false"
      :style="{ 
        width: !open && horizontal ? contentHeight : '',
        height: open || !horizontal ? contentHeight : '',
      }"
    >
      <slot :panel="panel" :open="open" />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, type PropType, inject, toRefs, type Ref, onMounted } from 'vue';
import type { CodeLayoutConfig, CodeLayoutContext, CodeLayoutPanelInternal } from './CodeLayout';
import { createMouseDragHandler } from './Composeable/MouseHandler';
import { checkDropPanelDefault, getCurrentDragPanel, usePanelDragOverDetector, usePanelDragger } from './Composeable/DragDrop';
import { usePanelMenuControl } from './Composeable/PanelMenu';
import CodeLayoutCollapseTitle from './CodeLayoutCollapseTitle.vue';

const emit = defineEmits([ 
  'update:open', 'update:resizeDragging',
  'toggleHandler',
]);

const props = defineProps({
  panel: {
    type: Object as PropType<CodeLayoutPanelInternal>,
    required: true,
  },
  selected: {
    type: Boolean,
    default: false,
  },
  /**
   * Only my self? Id true, do not show header
   */
  alone: {
    type: Boolean,
    default: false,
  },
  open: {
    type: Boolean,
    default: false,
  },
  dragResizeable: {
    type: Boolean,
    default: false,
  },
  dragResizeStartHandler: {
    type: Function as PropType<(panel: CodeLayoutPanelInternal, mousePx: number) => unknown>,
    default: null,
  },
  dragResizeHandler: {
    type: Function as PropType<(panel: CodeLayoutPanelInternal, data: unknown, mousePx: number) => void>,
    default: null,
  },
  horizontal: {
    type: Boolean,
    default: true,
  },
  collapsedSize: {
    type: Number,
    default: 0,
  },
  resizeDragging: {
    type: Boolean,
    default: false,
  },
  opening: {
    type: Boolean,
    default: false,
  },
});

const panelHeight = computed(() => {
  if (!props.open)
    return props.collapsedSize;
  return props.panel.size;
});

const contentHeight = computed(() => {
  if (!props.alone)
    return `calc(100% - ${layoutConfig.value.panelHeaderHeight}px)`;
  return `100%`;
}); 

const { horizontal, panel } = toRefs(props);
const element = ref<HTMLElement>();
const layoutConfig = inject('codeLayoutConfig') as Ref<CodeLayoutConfig>;
const context = inject('codeLayoutContext') as CodeLayoutContext;

//拖放面板处理函数

const {
  dragSelfState,
  handleDragStart,
  handleDragEnd,
} = usePanelDragger();

const {
  dragPanelState,
  dragLightBoxState,
  dragOverState,
  handleDragOver,
  handleDragEnter,
  handleDragLeave,
  handleDropPreCheck,
  resetDragOverState,
  resetDragState,
} = usePanelDragOverDetector(
  element, panel, horizontal,
  () => {},
  (e) => context.dragDropNonPanel(e, false, 'normal'),
  (dragPanel) => checkDropPanelDefault(dragPanel, panel.value, dragOverState)
);

function handleDrop(e: DragEvent) {
  const dropPanel = getCurrentDragPanel();
  if (dropPanel && dragOverState.value) {
    e.preventDefault();
    e.stopPropagation();
    context.dragDropToPanelNear(panel.value, dragOverState.value, dropPanel, 'normal');
  } else if (handleDropPreCheck(e)) {
    e.preventDefault();
    e.stopPropagation();
    context.dragDropNonPanel(e, true, 'normal', panel.value, dragOverState.value);
  }
  resetDragOverState();
  resetDragState();
}
function handleHeaderClick() {
  emit('toggleHandler', props.panel, !props.open);
}

//拖拽调整大小

let resizeDraggerData : any = null;
const resizeDragger = ref<HTMLElement>();
const resizeDraggingSelf = ref(false);
const resizeDragHandler = createMouseDragHandler({
  onDown(e) {
    if (resizeDragger.value) {
      resizeDraggerData = props.dragResizeStartHandler?.(
        props.panel, 
        (props.horizontal ? e.x : e.y)
      );

      if (resizeDraggerData === false)
        return false;

      emit('update:resizeDragging', true);
      resizeDraggingSelf.value = true;
      return true;
    }
    return false;
  },
  onMove(downPos, movedPos, e) {
    props.dragResizeHandler?.(
      props.panel, 
      resizeDraggerData,
      (props.horizontal ? e.x : e.y)
    );
  },
  onUp() {
    emit('update:resizeDragging', false);
    resizeDraggingSelf.value = false;
  },
});

//菜单处理
const {
  onContextMenu
} = usePanelMenuControl();

onMounted(() => {
  dragPanelState.value = false;
})
</script>

<style lang="scss">

.code-layout-panel {
  position: relative;
  flex-shrink: 0;
  flex-grow: 0;

  &.opening {
    transition: all ease-in-out 0.15s;
  }

  //状态控制
  &.open {
    > .code-layout-collapse {
      border-bottom-color: transparent;

      .arrow {
        transform: rotate(90deg) ;
      }
    }
  }
  &:focus {
    > .code-layout-collapse {
      border: 1px solid var(--code-layout-color-highlight);

      .actions {
        visibility: visible;
      }
    }
  }
  &.resizing {
    transition: none ease-in-out 0s;
  }
  &.resizing-self {

    .code-layout-collapse {
      cursor: inherit;
    }

    cursor: ns-resize;

    &.horizontal {
      cursor: ew-resize;
    }
  }
  &:hover {
    > .code-layout-collapse .actions {
      visibility: visible;
    }
  }
  &:first-child:not(:focus) > .code-layout-collapse {
    border-top-color: transparent;
  }

  //防止拖拽到内容上闪烁
  &.dragging:not(.dragging-self) * {
    pointer-events: none;
  }
  &.dragging.dragging-self {
    .collapse-title *, .content * {
      pointer-events: none;
    }
  }

  &::after, &::before {
    pointer-events: none;
  }

  //拖放状态处理
  &.drag-over-left, &.drag-over-up {
    //高亮框
    &.closed {
      background-color: var(--code-layout-color-background-mask-light);
    }
    &.open::after {
      background-color: var(--code-layout-color-background-mask-light);
      position: absolute;
      content: '';
      top: 0;
      left: 0;
      right: 0;
      height: 50%;
      z-index: 1;
    }
    &.horizontal.open::after {
      top: 0;
      left: 0;
      bottom: 0;
      right: unset;
      height: unset;
      width: 50%;
    }
    //高亮线条
    &.closed::before {
      position: absolute;
      content: '';
      z-index: 2;
      background-color: var(--code-layout-color-border-light);
      top: 0;
      left: 0;
      right: 0;
      height: 2px;
    } 
    &.horizontal.closed::before {
      right: unset;
      height: unset;
      width: 1px;
      bottom: 0;
    }
  }
  &.drag-over-right, &.drag-over-down {
    //高亮框
    &.closed {
      background-color: var(--code-layout-color-background-mask-light);
    }
    &.open::after {
      position: absolute;
      background-color: var(--code-layout-color-background-mask-light);
      content: '';
      top: 50%;
      left: 0;
      right: 0;
      height: 50%;
      z-index: 1;
    }
    &.horizontal.open::after {
      top: 0;
      left: 50%;
      bottom: 0;
      right: unset;
      height: unset;
      width: 50%;
    }
    //高亮线条
    &.closed::before {
      position: absolute;
      content: '';
      background-color: var(--code-layout-color-border-light);
      bottom: 0;
      left: 0;
      right: 0;
      height: 2px;
      z-index: 2;
    } 
    &.horizontal.closed::before {
      left: unset;
      height: unset;
      width: 1px;
      top: 0;
    }
  }
  &.drag-enter  {
    * {
      pointer-events: none;
    }
    > .drag-line {
      pointer-events: none;
      display: none;
    }
  }

  //水平状态特殊处理
  &.horizontal {
    > .drag-line {
      cursor: ew-resize;
      top: 0;
      left: calc(var(--code-layout-border-size-dragger) / 2 * -1);
      right: unset;
      bottom: 0;
      width: var(--code-layout-border-size-dragger);
      height: unset;
    }

    &:not(:first-child) {
      border-left: 1px solid var(--code-layout-color-border);
    }

    .code-layout-collapse {
      border-top: none;
      background-color: var(--code-layout-color-background-light);
    }

    &.closed {
      .code-layout-collapse {
        height: 100%;
        width: var(--code-layout-header-height);
        flex-direction: column;
        justify-content: flex-start;
        padding: 6px 0;

        .actions {
          display: none;
        }
        .arrow {
          margin-bottom: 5px;
        }
      }
      .collapse-title {
        flex-direction: column;
        justify-content: flex-start;
        
        span {
          display: none;
        }
      }
    }
  }

  //拖拽线条
  > .drag-line {
    position: absolute;
    left: 0;
    right: 0;
    top: calc(var(--code-layout-border-size-dragger) / 2 * -1);
    height: var(--code-layout-border-size-dragger);
    transition: background-color ease-in-out 0.4s;
    z-index: 10;
    cursor: ns-resize;

    &.active, &:hover {
      background-color: var(--code-layout-color-highlight);
    }
  }

  //内容区
  > .content {
    position: relative;
    overflow: hidden;
  }
}

</style>