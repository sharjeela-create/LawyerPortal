<script setup lang="ts">
import * as z from 'zod'
import { computed, onMounted, ref } from 'vue'
import { onBeforeRouteLeave } from 'vue-router'
import type { FormSubmitEvent } from '@nuxt/ui'

import { useAuth } from '../../composables/useAuth'
import { useAttorneyProfile } from '../../composables/useAttorneyProfile'
import UnsavedChangesModal from '../../components/settings/UnsavedChangesModal.vue'

const expertiseSchema = z.object({
  licensedStates: z.array(z.string()).min(1, 'At least one state is required'),
  primaryCity: z.string().min(2, 'Primary city is required'),
  countiesCovered: z.array(z.string()).optional(),
  federalCourts: z.string().optional(),
  primaryPracticeFocus: z.string().min(1, 'Practice focus is required'),
  injuryCategories: z.array(z.string()).min(1, 'At least one injury category is required'),
  exclusionaryCriteria: z.array(z.string()).optional(),
  minimumCaseValue: z.number().min(0).optional().or(z.literal(''))
})

type ExpertiseSchema = z.output<typeof expertiseSchema>

const auth = useAuth()
const attorneyProfile = useAttorneyProfile()
const saving = ref(false)
const toast = useToast()

const userId = computed(() => auth.state.value.user?.id ?? '')

const profile = attorneyProfile.draft as unknown as { value: Partial<ExpertiseSchema> }

const stateOptions = [
  'AL', 'AK', 'AZ', 'AR', 'CA', 'CO', 'CT', 'DE', 'FL', 'GA',
  'HI', 'ID', 'IL', 'IN', 'IA', 'KS', 'KY', 'LA', 'ME', 'MD',
  'MA', 'MI', 'MN', 'MS', 'MO', 'MT', 'NE', 'NV', 'NH', 'NJ',
  'NM', 'NY', 'NC', 'ND', 'OH', 'OK', 'OR', 'PA', 'RI', 'SC',
  'SD', 'TN', 'TX', 'UT', 'VT', 'VA', 'WA', 'WV', 'WI', 'WY'
]

const practiceFocusOptions = [
  'Personal Injury',
  'Medical Malpractice',
  'Workers Compensation',
  'Product Liability',
  'Wrongful Death',
  'Mass Torts',
  'Class Actions'
]

const injuryCategoryOptions = [
  'Auto Accidents',
  'Truck Accidents',
  'Motorcycle Accidents',
  'Pedestrian Accidents',
  'Slip and Fall',
  'Medical Malpractice',
  'Nursing Home Abuse',
  'Birth Injuries',
  'Workplace Injuries',
  'Construction Accidents',
  'Dog Bites',
  'Defective Products',
  'Toxic Exposure',
  'Brain Injuries',
  'Spinal Cord Injuries',
  'Burn Injuries',
  'Wrongful Death'
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

async function onSubmit(event: FormSubmitEvent<ExpertiseSchema>) {
  if (!userId.value) return

  saving.value = true
  try {
    await attorneyProfile.commitEditing(userId.value, [
      'licensedStates',
      'primaryCity',
      'countiesCovered',
      'federalCourts',
      'primaryPracticeFocus',
      'injuryCategories',
      'exclusionaryCriteria',
      'minimumCaseValue'
    ])
    
    toast.add({
      title: 'Success',
      description: 'Your expertise and jurisdiction settings have been updated.',
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
    id="expertise"
    :schema="expertiseSchema"
    :state="profile"
    @submit="onSubmit"
  >
    <UPageCard
      title="Expertise & Jurisdiction"
      description="Define your practice areas and geographic coverage for case matching."
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
            form="expertise"
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
            Geographic Coverage
          </h3>
        </div>

        <UFormField
          name="licensedStates"
          label="Licensed States"
          description="Select all states where you are licensed to practice"
          required
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInputMenu
            v-model="profile.licensedStates"
            :items="stateOptions"
            multiple
            searchable
            creatable
            placeholder="Select or type states"
            :disabled="disabled"
          />
        </UFormField>

        <USeparator />

        <UFormField
          name="primaryCity"
          label="Primary City/Metro Area"
          description="Your main practice location"
          required
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInput
            v-model="profile.primaryCity"
            placeholder="Los Angeles"
            autocomplete="off"
            :disabled="disabled"
          />
        </UFormField>

        <USeparator />

        <UFormField
          name="countiesCovered"
          label="Counties/Regions Covered"
          description="Specific counties or regions (leave empty for statewide)"
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInput
            v-model="profile.countiesCovered"
            placeholder="Enter counties separated by commas"
            autocomplete="off"
            :disabled="disabled"
          />
        </UFormField>

        <USeparator />

        <UFormField
          name="federalCourts"
          label="Federal Court Admissions"
          description="List any federal court admissions (optional)"
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UTextarea
            v-model="profile.federalCourts"
            :rows="3"
            placeholder="e.g., Central District of California, 9th Circuit Court of Appeals"
            autocomplete="off"
            :disabled="disabled"
          />
        </UFormField>
      </div>
    </UPageCard>

    <UPageCard variant="subtle">
      <div class="space-y-6">
        <div class="flex items-center justify-between">
          <h3 class="text-lg font-semibold">
            Case Specialization
          </h3>
        </div>

        <UFormField
          name="primaryPracticeFocus"
          label="Primary Practice Focus"
          description="Your main area of legal practice"
          required
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInputMenu
            v-model="profile.primaryPracticeFocus"
            :items="practiceFocusOptions"
            searchable
            creatable
            placeholder="Select or type practice focus"
            :disabled="disabled"
          />
        </UFormField>

        <USeparator />

        <UFormField
          name="injuryCategories"
          label="Specific Injury Categories"
          description="Select all types of cases you handle"
          required
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInputMenu
            v-model="profile.injuryCategories"
            :items="injuryCategoryOptions"
            multiple
            searchable
            creatable
            placeholder="Select or type categories"
            :disabled="disabled"
          />
        </UFormField>

        <USeparator />

        <UFormField
          name="exclusionaryCriteria"
          label="Exclusionary Criteria"
          description="Case types you do NOT handle (optional)"
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInput
            v-model="profile.exclusionaryCriteria"
            placeholder="e.g., Medical Malpractice, Class Actions"
            autocomplete="off"
            :disabled="disabled"
          />
        </UFormField>

        <USeparator />

        <UFormField
          name="minimumCaseValue"
          label="Minimum Case Value ($)"
          description="Minimum case value you will consider (optional)"
          class="flex max-sm:flex-col justify-between items-start gap-4"
        >
          <UInput
            v-model.number="profile.minimumCaseValue"
            type="number"
            min="0"
            placeholder="50000"
            autocomplete="off"
            :disabled="disabled"
          >
            <template #leading>
              <span class="text-dimmed">$</span>
            </template>
          </UInput>
        </UFormField>
      </div>
    </UPageCard>
  </UForm>
</template>
