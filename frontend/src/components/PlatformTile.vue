<template>
  <div
    class="tile"
    :data-platform="platform"
    :data-feature="platform"
    :class="{
      'is-loading': isLoading,
      'is-disabled': isDisabled,
      'is-ready': hasContent && !isLoading
    }"
    @click="onClick"
  >
    <div class="tile-icon" :class="iconClass">
      <img :src="icon" :alt="title" />
    </div>

    <h3>{{ title }}</h3>

    <p class="tile-hint">
      {{ hintText }}
    </p>
  </div>
</template>

<script setup>
const props = defineProps({
  platform: String,
  title: String,
  icon: String,
  iconClass: String,
  hintText: String,
  isLoading: Boolean,
  isDisabled: Boolean,
  hasContent: Boolean
})

const emit = defineEmits(['tile-click'])

function onClick() {
  emit('tile-click', props.platform)
}
</script>