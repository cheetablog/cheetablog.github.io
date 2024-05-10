<script setup lang="ts">
import { ref } from 'vue'
import ExternalVideo from './ExternalVideo.vue';

defineProps<{summary: string}>();

const isOpened = ref(false)

const handleToggle = () => {
    isOpened.value = true
}
</script>

<template>
    <details class="details custom-block" @toggle="handleToggle">
        <summary>{{ summary }}</summary>
        <ExternalVideo
            v-if="isOpened"
            v-bind="$attrs"
            showImmediately
            autoplay
            width="100%"
            controls
            playsinline="true"
            muted
            loop
        />
    </details>
</template>