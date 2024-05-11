<script setup lang="ts">
import { ref } from 'vue'

defineProps<{summary: string}>();

const isOpened = ref(false)

const handleToggle = () => {
    isOpened.value = true
}
</script>

<template>
    <details class="details custom-block" @toggle="handleToggle">
        <summary>{{ summary }}</summary>
        <slot v-if="isOpened" />
    </details>
</template>