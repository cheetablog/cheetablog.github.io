<script setup lang="ts">
import { ref } from 'vue';

const props = defineProps<{
    showImmediately?: boolean;
}>();

const isShow = ref(props.showImmediately)

const show = () => {
    isShow.value = true
}

defineExpose({ show })
</script>

<template>
<div class="media-placeholder-wrapper" :data-show="isShow" v-bind="$attrs">
    <div class="media-placeholder-content">
        <slot />
    </div>
</div>
</template>

<style>
.media-placeholder-wrapper {
    background-color: var(--vp-c-gray-1);
    width: 100%;
    border-radius: 11px;
}

.media-placeholder-content {
    opacity: 0;
    transition: opacity .2s ease;
}

.media-placeholder-content iframe,
.media-placeholder-content * {
    border-radius: 10px;
}

.media-placeholder-wrapper[data-show=true] .media-placeholder-content {
    opacity: 1;
}
</style>