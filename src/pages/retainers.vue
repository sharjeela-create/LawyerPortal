<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { useRouter } from 'vue-router'
import type { TableColumn } from '@nuxt/ui'

import { supabase } from '../lib/supabase'
import { useAuth } from '../composables/useAuth'

type DailyDealFlow = {
  id: string
  submission_id: string
  insured_name: string | null
  client_phone_number: string | null
  lead_vendor: string | null
  date: string | null
  status: string | null
  agent: string | null
  carrier: string | null
  created_at: string | null
}

const router = useRouter()
const auth = useAuth()

const loading = ref(false)
const error = ref<string | null>(null)
const query = ref('')

const rows = ref<DailyDealFlow[]>([])

const filteredRows = computed(() => {
  const q = query.value.trim().toLowerCase()
  if (!q) return rows.value

  return rows.value.filter((r) => {
    const haystack = [
      r.submission_id,
      r.insured_name ?? '',
      r.client_phone_number ?? '',
      r.lead_vendor ?? '',
      r.status ?? '',
      r.agent ?? '',
      r.carrier ?? ''
    ].join(' ').toLowerCase()

    return haystack.includes(q)
  })
})

const columns: TableColumn<DailyDealFlow>[] = [
  {
    accessorKey: 'date',
    header: 'Date'
  },
  {
    accessorKey: 'insured_name',
    header: 'Customer Name'
  },
  {
    accessorKey: 'client_phone_number',
    header: 'Phone Number'
  },
  {
    accessorKey: 'status',
    header: 'Status'
  },
  {
    accessorKey: 'agent',
    header: 'Assigned Lawyer'
  },
  {
    accessorKey: 'actions',
    header: 'Actions'
  }
]

const formatDate = (value: string | null) => {
  if (!value) return '—'
  // Prefer YYYY-MM-DD display like the UI screenshot
  return value.length >= 10 ? value.slice(0, 10) : value
}

const statusColor = (value: string | null) => {
  const normalized = (value ?? '').toLowerCase()
  if (normalized.includes('pending')) return 'warning' as const
  if (normalized.includes('approved') || normalized.includes('paid') || normalized.includes('complete')) return 'success' as const
  if (normalized.includes('rejected') || normalized.includes('failed') || normalized.includes('cancel')) return 'error' as const
  return 'neutral' as const
}

const load = async () => {
  loading.value = true
  error.value = null

  try {
    await auth.init()
    
    const userId = auth.state.value.user?.id
    const userRole = auth.state.value.profile?.role
    
    let queryBuilder = supabase
      .from('daily_deal_flow')
      .select('id,submission_id,insured_name,client_phone_number,lead_vendor,date,status,agent,carrier,created_at')

    // If user is a lawyer, only show leads assigned to them
    if (userRole === 'lawyer' && userId) {
      queryBuilder = queryBuilder.eq('assigned_attorney_id', userId)
    } else if (!userRole) {
      // If no role, filter by lead vendor matching user's center
      // Get user's center first
      const { data: userData } = await supabase
        .from('app_users')
        .select('center_id')
        .eq('user_id', userId || '')
        .maybeSingle()
      
      if (userData?.center_id) {
        // Get center's lead vendor
        const { data: centerData } = await supabase
          .from('centers')
          .select('lead_vendor')
          .eq('id', userData.center_id)
          .maybeSingle()
        
        if (centerData?.lead_vendor) {
          queryBuilder = queryBuilder.eq('lead_vendor', centerData.lead_vendor)
        }
      }
    }
    // Admin and agent roles see all leads (no filter)

    const { data, error: supaError } = await queryBuilder
      .order('created_at', { ascending: false })
      .limit(250)

    if (supaError) throw supaError

    rows.value = (data ?? []) as DailyDealFlow[]
  } catch (e) {
    const msg = e instanceof Error ? e.message : 'Failed to load retainers'
    error.value = msg
  } finally {
    loading.value = false
  }
}

onMounted(load)

const openRow = (row: DailyDealFlow) => {
  router.push(`/retainers/${row.id}`)
}
</script>

<template>
  <UDashboardPanel id="retainers">
    <template #header>
      <UDashboardNavbar title="Retainers">
        <template #leading>
          <UDashboardSidebarCollapse />
        </template>

        <template #right>
          <UButton
            color="neutral"
            variant="outline"
            icon="i-lucide-refresh-cw"
            :loading="loading"
            @click="load"
          >
            Refresh
          </UButton>
        </template>
      </UDashboardNavbar>
    </template>

    <template #body>
      <div class="flex flex-wrap items-center justify-between gap-3">
        <UInput
          v-model="query"
          class="max-w-md"
          icon="i-lucide-search"
          placeholder="Search by name, phone..."
        />

        <UBadge
          variant="subtle"
          :label="`${filteredRows.length} leads`"
        />
      </div>

      <UAlert
        v-if="error"
        class="mt-4"
        color="error"
        variant="subtle"
        title="Unable to load retainers"
        :description="error"
      />

      <UPageCard variant="subtle" class="mt-4">
        <UTable
          class="mt-0"
          :loading="loading"
          :data="filteredRows"
          :columns="columns"
          :ui="{
            base: 'table-fixed border-separate border-spacing-0',
            thead: '[&>tr]:bg-elevated/50 [&>tr]:after:content-none',
            tbody: '[&>tr]:last:[&>td]:border-b-0',
            th: 'first:rounded-l-lg last:rounded-r-lg border-y border-default first:border-l last:border-r',
            td: 'border-b border-default'
          }"
        >
          <template #date-cell="{ row }">
            <span class="text-sm text-white/80">{{ formatDate(row.original.date) }}</span>
          </template>

          <template #insured_name-cell="{ row }">
            <span class="text-sm text-white/90">{{ row.original.insured_name ?? '—' }}</span>
          </template>

          <template #client_phone_number-cell="{ row }">
            <span class="text-sm text-white/80">{{ row.original.client_phone_number ?? '—' }}</span>
          </template>

          <template #status-cell="{ row }">
            <UBadge
              variant="subtle"
              :color="statusColor(row.original.status)"
              class="text-xs"
            >
              {{ row.original.status ?? '—' }}
            </UBadge>
          </template>

          <template #agent-cell="{ row }">
            <span class="text-sm text-white/80">{{ row.original.agent ?? '—' }}</span>
          </template>

          <template #actions-cell="{ row }">
            <div class="flex justify-end">
              <UButton
                size="xs"
                color="neutral"
                variant="outline"
                icon="i-lucide-eye"
                label="View"
                @click="openRow(row.original)"
              />
            </div>
          </template>
        </UTable>
      </UPageCard>
    </template>
  </UDashboardPanel>
</template>
