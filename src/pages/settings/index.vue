<script setup lang="ts">
import * as z from 'zod'
import { computed, onMounted, reactive, ref, watch } from 'vue'
import type { FormSubmitEvent } from '@nuxt/ui'

import { useAuth } from '../../composables/useAuth'
import { supabase } from '../../lib/supabase'

const profileSchema = z.object({
  name: z.string().min(2, 'Too short'),
  email: z.string().email('Invalid email')
})

type ProfileSchema = z.output<typeof profileSchema>

const auth = useAuth()
const saving = ref(false)
const isEditing = ref(false)

const profile = reactive<Partial<ProfileSchema>>({
  name: '',
  email: ''
})

const disabled = computed(() => !isEditing.value)

const hydrateFromAuth = () => {
  const p = auth.state.value.profile
  profile.name = p?.display_name ?? ''
  profile.email = p?.email ?? auth.state.value.user?.email ?? ''
}

onMounted(async () => {
  await auth.init()
  hydrateFromAuth()
})

watch(
  () => auth.state.value.profile,
  () => {
    hydrateFromAuth()
  }
)

const toast = useToast()
async function onSubmit(event: FormSubmitEvent<ProfileSchema>) {
  const token = auth.state.value.session?.access_token ?? ''
  if (!token) return

  saving.value = true
  try {
    const { error } = await supabase
      .from('app_users')
      .update({ display_name: event.data.name })
      .eq('user_id', auth.state.value.user?.id)

    if (error) throw error

    await auth.refreshProfile()
    toast.add({
      title: 'Success',
      description: 'Your settings have been updated.',
      icon: 'i-lucide-check',
      color: 'success'
    })

    isEditing.value = false
  } catch (err) {
    const msg = err instanceof Error ? err.message : 'Unable to update profile'
    toast.add({
      title: 'Error',
      description: msg,
      icon: 'i-lucide-x',
      color: 'error'
    })
  } finally {
    saving.value = false
  }
}

const startEditing = () => {
  isEditing.value = true
}

const cancelEditing = () => {
  hydrateFromAuth()
  isEditing.value = false
}
</script>

<template>
  <UForm
    id="settings"
    :schema="profileSchema"
    :state="profile"
    @submit="onSubmit"
  >
    <UPageCard
      title="Profile"
      description="These informations will be displayed publicly."
      variant="naked"
      orientation="horizontal"
      class="mb-4"
    >
      <div class="flex items-center gap-2 w-fit lg:ms-auto">
        <UButton
          v-if="!isEditing"
          label="Edit"
          color="neutral"
          variant="outline"
          class="w-fit"
          @click="startEditing"
        />
        <template v-else>
          <UButton
            form="settings"
            label="Save changes"
            color="neutral"
            type="submit"
            :loading="saving"
            class="w-fit"
          />
          <UButton
            label="Cancel"
            color="neutral"
            variant="ghost"
            class="w-fit"
            @click="cancelEditing"
          />
        </template>
      </div>
    </UPageCard>

    <UPageCard variant="subtle">
      <UFormField
        name="name"
        label="Name"
        description="Will appear on receipts, invoices, and other communication."
        required
        class="flex max-sm:flex-col justify-between items-start gap-4"
      >
        <UInput
          v-model="profile.name"
          autocomplete="off"
          :disabled="disabled"
        />
      </UFormField>
      <USeparator />
      <UFormField
        name="email"
        label="Email"
        description="Used to sign in, for email receipts and product updates."
        required
        class="flex max-sm:flex-col justify-between items-start gap-4"
      >
        <UInput
          v-model="profile.email"
          type="email"
          autocomplete="off"
          disabled
        />
      </UFormField>
    </UPageCard>
  </UForm>
</template>
