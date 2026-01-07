<script setup lang="ts">
import { computed } from 'vue'

interface ProfileData {
  // Tab 1: General Information
  fullName?: string
  firmName?: string
  barNumber?: string
  bio?: string
  yearsExperience?: number | string
  languages?: string[]
  directPhone?: string
  officeAddress?: string
  websiteUrl?: string
  preferredContact?: string
  assistantName?: string
  assistantEmail?: string
  
  // Tab 2: Expertise & Jurisdiction
  licensedStates?: string[]
  primaryCity?: string
  countiesCovered?: string[]
  federalCourts?: string
  primaryPracticeFocus?: string
  injuryCategories?: string[]
  exclusionaryCriteria?: string[]
  minimumCaseValue?: number | string
  
  // Tab 3: Capacity & Performance
  availabilityStatus?: string
  firmSize?: string
  caseManagementSoftware?: string
  insuranceCarriers?: string[]
  litigationStyle?: number
  largestSettlement?: number | string
  avgTimeToClose?: string
}

const props = defineProps<{
  profileData?: ProfileData
}>()

const requiredFields = [
  'fullName',
  'firmName',
  'barNumber',
  'languages',
  'directPhone',
  'officeAddress',
  'licensedStates',
  'primaryCity',
  'primaryPracticeFocus',
  'injuryCategories',
  'availabilityStatus'
]

const optionalFields = [
  'bio',
  'yearsExperience',
  'websiteUrl',
  'preferredContact',
  'assistantName',
  'assistantEmail',
  'countiesCovered',
  'federalCourts',
  'exclusionaryCriteria',
  'minimumCaseValue',
  'firmSize',
  'caseManagementSoftware',
  'insuranceCarriers',
  'litigationStyle',
  'largestSettlement',
  'avgTimeToClose'
]

const completionPercentage = computed(() => {
  if (!props.profileData) return 0
  
  let filledRequired = 0
  let filledOptional = 0
  
  requiredFields.forEach(field => {
    const value = props.profileData?.[field as keyof ProfileData]
    if (value !== undefined && value !== null && value !== '' && 
        (!Array.isArray(value) || value.length > 0)) {
      filledRequired++
    }
  })
  
  optionalFields.forEach(field => {
    const value = props.profileData?.[field as keyof ProfileData]
    if (value !== undefined && value !== null && value !== '' && 
        (!Array.isArray(value) || value.length > 0)) {
      filledOptional++
    }
  })
  
  const requiredWeight = 0.7
  const optionalWeight = 0.3
  
  const requiredScore = (filledRequired / requiredFields.length) * requiredWeight
  const optionalScore = (filledOptional / optionalFields.length) * optionalWeight
  
  return Math.round((requiredScore + optionalScore) * 100)
})

const completionColor = computed(() => {
  const pct = completionPercentage.value
  if (pct >= 80) return 'success'
  if (pct >= 50) return 'warning'
  return 'error'
})

const completionMessage = computed(() => {
  const pct = completionPercentage.value
  if (pct === 100) return 'Your profile is complete!'
  if (pct >= 80) return 'Almost there! Just a few more details.'
  if (pct >= 50) return 'Good progress. Keep going!'
  return 'Let\'s complete your profile to maximize case opportunities.'
})
</script>

<template>
  <div class="mb-6 p-4 rounded-lg border border-default bg-elevated/50">
    <div class="flex items-center justify-between mb-3">
      <div>
        <h3 class="text-sm font-semibold">
          Profile Completion
        </h3>
        <p class="text-xs text-dimmed mt-1">
          {{ completionMessage }}
        </p>
      </div>
      <div class="text-right">
        <span class="text-2xl font-bold" :class="`text-${completionColor}`">
          {{ completionPercentage }}%
        </span>
      </div>
    </div>
    <div class="h-2 w-full rounded-full bg-default overflow-hidden">
      <div
        class="h-full rounded-full transition-none"
        :class="{
          'bg-success': completionColor === 'success',
          'bg-warning': completionColor === 'warning',
          'bg-error': completionColor === 'error'
        }"
        :style="{ width: `${completionPercentage}%` }"
      />
    </div>
    
    <div class="mt-3 flex items-center gap-4 text-xs text-dimmed">
      <div class="flex items-center gap-1">
        <UIcon name="i-lucide-check-circle" class="size-4" />
        <span>{{ requiredFields.filter(f => {
          const value = profileData?.[f as keyof ProfileData]
          return value !== undefined && value !== null && value !== '' && 
                 (!Array.isArray(value) || value.length > 0)
        }).length }} / {{ requiredFields.length }} Required</span>
      </div>
      <div class="flex items-center gap-1">
        <UIcon name="i-lucide-star" class="size-4" />
        <span>{{ optionalFields.filter(f => {
          const value = profileData?.[f as keyof ProfileData]
          return value !== undefined && value !== null && value !== '' && 
                 (!Array.isArray(value) || value.length > 0)
        }).length }} / {{ optionalFields.length }} Optional</span>
      </div>
    </div>
  </div>
</template>
