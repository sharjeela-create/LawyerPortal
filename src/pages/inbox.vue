<script setup lang="ts">
import { ref } from 'vue'

const unreadCount = ref(0)
</script>

<template>
  <UDashboardPanel
    id="inbox-1"
    :default-size="25"
    :min-size="20"
    :max-size="30"
    resizable
  >
    <UDashboardNavbar title="Inbox">
      <template #leading>
        <UDashboardSidebarCollapse />
      </template>
      <template #trailing>
        <UBadge :label="unreadCount" variant="subtle" />
      </template>
    </UDashboardNavbar>

    <div class="flex flex-1 items-center justify-center p-6">
      <div class="flex flex-col items-center gap-3 text-center">
        <UIcon name="i-lucide-inbox" class="size-16 text-dimmed" />
        <p class="text-sm text-dimmed">Inbox is currently empty.</p>
      </div>
    </div>
  </UDashboardPanel>
</template>
