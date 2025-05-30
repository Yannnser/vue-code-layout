<template>
  <SimpleTooltip
    :content="panel.tooltip"
    :direction="'top'"
  >
    <div
      ref="container"
      :class="[
        'tab-item',
        active ? 'active' : '',
        dragLightBoxState ? 'drag-enter' : '',
        `drag-over-${dragOverState}`,
      ]"
      :draggable="panel.draggable"
      @dragstart="handleDragStart(panel, $event)"
      @dragend="handleDragEnd"
      @dragover="handleDragOver"
      @dragleave="handleDragLeave"
      @dragenter="handleDragEnter"
      @drop="handleDrop"
      @contextmenu="onContextMenu(panel, $event)"
    >
      <span v-if="tabStyle == 'text' || tabStyle == 'text-bottom'" class="title">{{ panel.title }}</span>
      <span v-if="tabStyle == 'icon' || tabStyle == 'icon-bottom'" class="icon">
        <CodeLayoutVNodeStringRender :content="panel.iconSmall || panel.iconLarge || panel.title" />
      </span>
      <span v-if="panel.badge" class="badge">
        <CodeLayoutVNodeStringRender :content="panel.badge" />
      </span>
    </div>
  </SimpleTooltip>
</template>

<script setup lang="ts">
import { ref, type PropType, toRefs, inject } from 'vue';
import { checkDropPanelDefault, getCurrentDragPanel, usePanelDragOverDetector, usePanelDragger } from './Composeable/DragDrop';
import CodeLayoutVNodeStringRender from './Components/CodeLayoutVNodeStringRender.vue';
import type { CodeLayoutContext, CodeLayoutPanelInternal, CodeLayoutPanelTabStyle } from './CodeLayout';
import { usePanelMenuControl } from './Composeable/PanelMenu';
import SimpleTooltip from './Components/SimpleTooltip.vue';

const emit = defineEmits([
  'focusSelf'
]);

const props = defineProps({
  tabStyle: {
    type: String as PropType<CodeLayoutPanelTabStyle>,
    default: 'none',
  },
  active: {
    type: Boolean,
    default: true,
  },
  panel: {
    type: Object as PropType<CodeLayoutPanelInternal>,
    required: true,
  },
});
const { panel } = toRefs(props);
const horizontal = ref(true);
const container = ref<HTMLElement>();
const context = inject('codeLayoutContext') as CodeLayoutContext;

const {
  handleDragStart,
  handleDragEnd,
} = usePanelDragger();

const {
  dragLightBoxState,
  dragEnterState,
  dragOverState,
  handleDragOver,
  handleDragEnter,
  handleDragLeave,
  handleDropPreCheck,
  resetDragState,
  resetDragOverState,
} = usePanelDragOverDetector(
  container, panel, horizontal, 
  () => emit('focusSelf'),
  (e) => context.dragDropNonPanel(e, false, 'tab-header'),
  (dragPanel) => checkDropPanelDefault(dragPanel, panel.value, dragOverState)
);

function handleDrop(e: DragEvent) {
  const dropPanel = getCurrentDragPanel();
  if (dropPanel && dragOverState.value) {
    e.preventDefault();
    e.stopPropagation();
    context.dragDropToPanelNear(panel.value, dragOverState.value, dropPanel, 'tab-header');
  } else if (handleDropPreCheck(e)) {
    e.preventDefault();
    e.stopPropagation();
    context.dragDropNonPanel(e, true, 'tab-header', panel.value, dragOverState.value);
  }
  resetDragOverState();
  resetDragState();
}

//菜单处理
const {
  onContextMenu
} = usePanelMenuControl();
</script>