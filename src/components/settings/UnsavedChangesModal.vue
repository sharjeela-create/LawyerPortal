<script setup lang="ts">
const props = defineProps<{
  open: boolean
}>()

const emit = defineEmits<{
  (e: 'update:open', value: boolean): void
  (e: 'confirm'): void
  (e: 'cancel'): void
}>()

const handleUpdateOpen = (value: boolean) => {
  emit('update:open', value)
  if (!value) {
    emit('cancel')
  }
}
</script>

<template>
  <UModal
    :open="props.open"
    title="Unsaved changes"
    :dismissible="false"
    @update:open="handleUpdateOpen"
  >
    <template #body>
      <div class="space-y-4">
        <p class="text-sm text-white/80">
          You have unsaved changes. Do you want to discard them?
        </p>

        <div class="flex justify-end gap-2">
          <UButton
            color="neutral"
            variant="ghost"
            @click="emit('cancel')"
          >
            Stay
          </UButton>
          <UButton
            color="error"
            @click="emit('confirm')"
          >
            Discard
          </UButton>
        </div>
      </div>
    </template>
  </UModal>
</template>
