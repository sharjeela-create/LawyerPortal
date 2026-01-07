<script setup lang="ts">
import { ref, computed } from 'vue'
import type { DropdownMenuItem } from '@nuxt/ui'
import { useColorMode } from '@vueuse/core'
import { BRANDING } from '../lib/branding'

defineProps<{
  collapsed?: boolean
}>()

const colorMode = useColorMode()

const logoSrc = computed(() => {
  return colorMode.value === 'dark' ? '/assets/logo-white.png' : '/assets/logo-black.png'
})

const teams = ref([{
  label: 'Lawyer Portal',
  avatar: {
    src: BRANDING.logo,
    alt: 'Lawyer Portal'
  }
}])
const selectedTeam = ref(teams.value[0])

const items = computed<DropdownMenuItem[][]>(() => {
  return [teams.value.map(team => ({
    ...team,
    onSelect() {
      selectedTeam.value = team
    }
  })), [{
    label: 'Create team',
    icon: 'i-lucide-circle-plus'
  }, {
    label: 'Manage teams',
    icon: 'i-lucide-cog'
  }]]
})
</script>

<template>
  <UDropdownMenu
    :items="items"
    :content="{ align: 'center', collisionPadding: 12 }"
    :ui="{ content: collapsed ? 'w-40' : 'w-(--reka-dropdown-menu-trigger-width)' }"
  >
    <UButton
      color="neutral"
      variant="ghost"
      block
      :square="collapsed"
      class="data-[state=open]:bg-elevated"
      :class="[collapsed ? 'py-2' : 'py-3']"
    >
      <img
        :src="logoSrc"
        :alt="selectedTeam.avatar.alt"
        :class="collapsed ? 'h-7 w-auto' : 'h-8 w-auto max-w-44'"
      >
    </UButton>
  </UDropdownMenu>
</template>
