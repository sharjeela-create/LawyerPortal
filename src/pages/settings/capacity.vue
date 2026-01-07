<script setup lang="ts">
import * as z from 'zod'
import { computed, onMounted, ref } from 'vue'
import { onBeforeRouteLeave } from 'vue-router'
import type { FormSubmitEvent } from '@nuxt/ui'

import { useAuth } from '../../composables/useAuth'
import { useAttorneyProfile } from '../../composables/useAttorneyProfile'
import UnsavedChangesModal from '../../components/settings/UnsavedChangesModal.vue'

const capacitySchema = z.object({
  availabilityStatus: z.enum(['accepting', 'at_capacity', 'on_leave']),
  firmSize: z.enum(['solo', 'small', 'medium', 'large']).optional(),
  caseManagementSoftware: z.string().optional(),
  insuranceCarriers: z.array(z.string()).optional(),
  litigationStyle: z.number().min(1).max(5).optional(),
  largestSettlement: z.number().min(0).optional().or(z.literal('')),
  avgTimeToClose: z.string().optional()
})

type CapacitySchema = z.output<typeof capacitySchema>

const auth = useAuth()
const attorneyProfile = useAttorneyProfile()
const saving = ref(false)
const toast = useToast()

const userId = computed(() => auth.state.value.user?.id ?? '')

const profile = attorneyProfile.draft as unknown as { value: Partial<CapacitySchema> }

const availabilityOptions = [
  { label: 'Accepting Cases (Active)', value: 'accepting', description: 'Available for new case leads' },
  { label: 'At Capacity (Paused)', value: 'at_capacity', description: 'Temporarily not accepting new cases' },
  { label: 'On Leave', value: 'on_leave', description: 'Not available for case assignments' }
]

const firmSizeOptions = [
  'Solo',
  'Small (2-10)',
  'Medium (11-50)',
  'Large (50+)'
]

const caseManagementOptions = [
  'Clio',
  'MyCase',
  'PracticePanther',
  'Smokeball',
  'CASEpeer',
  'Filevine',
  'LawRuler',
  'Other',
  'None'
]

const insuranceCarrierOptions = [
  'State Farm',
  'GEICO',
  'Progressive',
  'Allstate',
  'USAA',
  'Liberty Mutual',
  'Farmers',
  'Nationwide',
  'Travelers',
  'American Family'
]

const timeToCloseOptions = [
  'Less than 6 months',
  '6-12 months',
  '12-18 months',
  '18-24 months',
  'Over 24 months'
]

onMounted(async () => {
  await auth.init()
  if (userId.value) {
    await attorneyProfile.loadProfile(userId.value)
  }

  if (attorneyProfile.isEditing.value && !attorneyProfile.isDirty.value) {
    attorneyProfile.cancelEditing()
  }
})

async function onSubmit(event: FormSubmitEvent<CapacitySchema>) {
  if (!userId.value) return

  saving.value = true
  try {
    await attorneyProfile.commitEditing(userId.value, [
      'availabilityStatus',
      'firmSize',
      'caseManagementSoftware',
      'insuranceCarriers',
      'litigationStyle',
      'largestSettlement',
      'avgTimeToClose'
    ])
    
    toast.add({
      title: 'Success',
      description: 'Your capacity and performance settings have been updated.',
      icon: 'i-lucide-check',
      color: 'success'
    })
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

const disabled = computed(() => !attorneyProfile.isEditing.value)

const isEditing = computed(() => attorneyProfile.isEditing.value)

const cancelEditing = () => {
  attorneyProfile.cancelEditing()
}

const startEditing = () => {
  attorneyProfile.startEditing()
}

const unsavedOpen = ref(false)
const pendingNav = ref<null | (() => void)>(null)

const confirmLeave = () => {
  unsavedOpen.value = true
}

const handleConfirmDiscard = () => {
  attorneyProfile.cancelEditing()
  unsavedOpen.value = false
  const go = pendingNav.value
  pendingNav.value = null
  go?.()
}

const handleStay = () => {
  unsavedOpen.value = false
  pendingNav.value = null
}

onBeforeRouteLeave((to, from, next) => {
  if (attorneyProfile.isEditing.value && attorneyProfile.isDirty.value) {
    pendingNav.value = () => next()
    confirmLeave()
    next(false)
    return
  }

  next()
})
</script>

<template>
  <UnsavedChangesModal
    :open="unsavedOpen"
    @update:open="(v) => { unsavedOpen = v }"
    @confirm="handleConfirmDiscard"
    @cancel="handleStay"
  />

  <UForm
    id="capacity"
    :schema="capacitySchema"
    :state="profile"
    @submit="onSubmit"
  >
    <UPageCard
      title="Capacity & Performance"
      description="Manage your availability status and performance metrics."
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
            form="capacity"
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

    <UPageCard variant="subtle" class="mb-6">
      <div class="space-y-6">
        <div class="flex items-center justify-between">
          <h3 class="text-lg font-semibold">
            Real-Time Status
          </h3>
        </div>

        <UFormField
          name="availabilityStatus"
          label="Availability Status"
          description="Control whether you receive new case leads"
          required
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <URadioGroup
            v-model="profile.availabilityStatus"
            :options="availabilityOptions"
            :disabled="disabled"
          />
        </UFormField>

        <USeparator />

        <UFormField
          name="firmSize"
          label="Firm Size"
          description="Size of your law firm (optional)"
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInputMenu
            v-model="profile.firmSize"
            :items="firmSizeOptions"
            searchable
            creatable
            placeholder="Select or type firm size"
            :disabled="disabled"
          />
        </UFormField>

        <USeparator />

        <UFormField
          name="caseManagementSoftware"
          label="Case Management Software"
          description="Software you use to manage cases (optional)"
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInputMenu
            v-model="profile.caseManagementSoftware"
            :items="caseManagementOptions"
            searchable
            creatable
            placeholder="Select or type software"
            :disabled="disabled"
          />
        </UFormField>
      </div>
    </UPageCard>

    <UPageCard variant="subtle">
      <div class="space-y-6">
        <div class="flex items-center justify-between">
          <h3 class="text-lg font-semibold">
            Performance Metrics
          </h3>
          <span class="text-sm text-dimmed">Self-Reported</span>
        </div>

        <UFormField
          name="insuranceCarriers"
          label="Insurance Carriers Handled"
          description="Select carriers you have experience with (optional)"
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInputMenu
            v-model="profile.insuranceCarriers"
            :items="insuranceCarrierOptions"
            multiple
            searchable
            creatable
            placeholder="Select or type carriers"
            :disabled="disabled"
          />
        </UFormField>

        <USeparator />

        <UFormField
          name="litigationStyle"
          label="Litigation Style"
          description="Your approach to case resolution (optional)"
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <div class="w-full space-y-2">
            <URange
              v-model="profile.litigationStyle"
              :min="1"
              :max="5"
              :step="1"
              :disabled="disabled"
            />
            <div class="flex justify-between text-xs text-dimmed">
              <span>Settlement Focused</span>
              <span>Balanced</span>
              <span>Aggressive Trial</span>
            </div>
          </div>
        </UFormField>

        <USeparator />

        <UFormField
          name="largestSettlement"
          label="Largest Settlement Amount"
          description="Your largest case settlement or verdict (optional)"
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInput
            v-model.number="profile.largestSettlement"
            type="number"
            min="0"
            placeholder="1000000"
            autocomplete="off"
            :disabled="disabled"
          >
            <template #leading>
              <span class="text-dimmed">$</span>
            </template>
          </UInput>
        </UFormField>

        <USeparator />

        <UFormField
          name="avgTimeToClose"
          label="Average Time to Close"
          description="Typical duration from intake to resolution (optional)"
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInputMenu
            v-model="profile.avgTimeToClose"
            :items="timeToCloseOptions"
            searchable
            creatable
            placeholder="Select or type time"
            :disabled="disabled"
          />
        </UFormField>
      </div>
    </UPageCard>
  </UForm>
</template>
