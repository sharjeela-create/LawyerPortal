<script setup lang="ts">
import { computed } from 'vue'
import { useRoute } from 'vue-router'
import type { NavigationMenuItem } from '@nuxt/ui'
import ProfileCompletionMeter from '../components/settings/ProfileCompletionMeter.vue'
import { useAttorneyProfile } from '../composables/useAttorneyProfile'

const route = useRoute()
const attorneyProfile = useAttorneyProfile()

const attorneyProfileData = computed(() => attorneyProfile.state.value)

const links = [[{
  label: 'General',
  icon: 'i-lucide-user',
  to: '/settings',
  exact: true
}], [{
  label: 'Attorney Profile',
  icon: 'i-lucide-briefcase',
  to: '/settings/attorney-profile',
  exact: true
}], [{
  label: 'Expertise & Jurisdiction',
  icon: 'i-lucide-map-pin',
  to: '/settings/expertise',
  exact: true
}], [{
  label: 'Capacity & Performance',
  icon: 'i-lucide-activity',
  to: '/settings/capacity',
  exact: true
}]] satisfies NavigationMenuItem[][]

const showCompletionMeter = computed(() => {
  return route.path !== '/settings' && route.path.startsWith('/settings/')
})
</script>

<template>
  <UDashboardPanel id="settings" :ui="{ body: 'lg:py-12' }">
    <template #header>
      <UDashboardNavbar title="Settings">
        <template #leading>
          <UDashboardSidebarCollapse />
        </template>
      </UDashboardNavbar>

      <UDashboardToolbar>
        <!-- NOTE: The `-mx-1` class is used to align with the `DashboardSidebarCollapse` button here. -->
        <UNavigationMenu :items="links" highlight class="-mx-1 flex-1" />
      </UDashboardToolbar>
    </template>

    <template #body>
      <div class="flex flex-col gap-4 sm:gap-6 lg:gap-12 w-full">
        <ProfileCompletionMeter v-if="showCompletionMeter" :profile-data="attorneyProfileData" />
        <RouterView />
      </div>
    </template>
  </UDashboardPanel>
</template>
