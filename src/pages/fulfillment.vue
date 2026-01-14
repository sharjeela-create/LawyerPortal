<script setup lang="ts">
import { computed, ref } from 'vue'

type StageKey = 'returned_back' | 'dropped_retainers' | 'signed_retainers'

const STAGES: { key: StageKey, label: string }[] = [
  { key: 'returned_back', label: 'Returned Back' },
  { key: 'dropped_retainers', label: 'Dropped Retainers' },
  { key: 'signed_retainers', label: 'Signed Retainers' }
]

type FulfillmentOrder = {
  id: string
  date: string
  clientName: string
  phone: string
  state: string
  caseValue: number
  stage: StageKey
  reason?: string
  signedDate?: string
}

const loading = ref(false)
const query = ref('')
const selectedStage = ref<'all' | StageKey>('all')

const mockOrders = ref<FulfillmentOrder[]>([
  { id: 'F-1001', date: '2024-01-15', clientName: 'John Smith', phone: '555-0101', state: 'CA', caseValue: 25000, stage: 'signed_retainers', signedDate: '2024-01-15' },
  { id: 'F-1002', date: '2024-01-15', clientName: 'Jane Wilson', phone: '555-0102', state: 'TX', caseValue: 30000, stage: 'signed_retainers', signedDate: '2024-01-15' },
  { id: 'F-1003', date: '2024-01-14', clientName: 'Robert Johnson', phone: '555-0103', state: 'FL', caseValue: 18000, stage: 'dropped_retainers', reason: 'Client unresponsive' },
  { id: 'F-1004', date: '2024-01-14', clientName: 'Alice Williams', phone: '555-0104', state: 'NY', caseValue: 22000, stage: 'dropped_retainers', reason: 'Case value too low' },
  { id: 'F-1005', date: '2024-01-13', clientName: 'Michael Brown', phone: '555-0105', state: 'PA', caseValue: 35000, stage: 'signed_retainers', signedDate: '2024-01-13' },
  { id: 'F-1006', date: '2024-01-13', clientName: 'Emily Davis', phone: '555-0106', state: 'IL', caseValue: 28000, stage: 'signed_retainers', signedDate: '2024-01-13' },
  { id: 'F-1007', date: '2024-01-12', clientName: 'Daniel Miller', phone: '555-0107', state: 'OH', caseValue: 15000, stage: 'returned_back', reason: 'Incomplete documentation' },
  { id: 'F-1008', date: '2024-01-12', clientName: 'Sophia Garcia', phone: '555-0108', state: 'GA', caseValue: 20000, stage: 'returned_back', reason: 'Missing medical records' },
  { id: 'F-1009', date: '2024-01-11', clientName: 'William Martinez', phone: '555-0109', state: 'NC', caseValue: 32000, stage: 'signed_retainers', signedDate: '2024-01-11' },
  { id: 'F-1010', date: '2024-01-11', clientName: 'Olivia Anderson', phone: '555-0110', state: 'MI', caseValue: 26000, stage: 'signed_retainers', signedDate: '2024-01-11' },
  { id: 'F-1011', date: '2024-01-10', clientName: 'James Taylor', phone: '555-0111', state: 'WA', caseValue: 19000, stage: 'dropped_retainers', reason: 'Client declined' },
  { id: 'F-1012', date: '2024-01-10', clientName: 'Emma Thomas', phone: '555-0112', state: 'AZ', caseValue: 17000, stage: 'returned_back', reason: 'Verification needed' }
])

const filteredOrders = computed(() => {
  const q = query.value.trim().toLowerCase()
  return mockOrders.value.filter((o) => {
    if (selectedStage.value !== 'all' && o.stage !== selectedStage.value) return false
    if (!q) return true
    const stageLabel = STAGES.find(s => s.key === o.stage)?.label ?? ''
    const haystack = [o.id, o.clientName, o.phone, o.state, stageLabel].join(' ').toLowerCase()
    return haystack.includes(q)
  })
})

const ordersByStage = computed(() => {
  const grouped = new Map<StageKey, FulfillmentOrder[]>()
  STAGES.forEach((s) => grouped.set(s.key, []))
  filteredOrders.value.forEach((o) => {
    const arr = grouped.get(o.stage)
    if (arr) arr.push(o)
  })
  return grouped
})

const totalOrders = computed(() => mockOrders.value.length)
const signedCount = computed(() => mockOrders.value.filter(o => o.stage === 'signed_retainers').length)
const droppedCount = computed(() => mockOrders.value.filter(o => o.stage === 'dropped_retainers').length)
const returnedCount = computed(() => mockOrders.value.filter(o => o.stage === 'returned_back').length)

const formatMoney = (n: number) => {
  try {
    return new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 }).format(n)
  } catch {
    return `$${n}`
  }
}

const getStageColor = (stage: StageKey) => {
  if (stage === 'signed_retainers') return 'success'
  if (stage === 'dropped_retainers') return 'error'
  return 'warning'
}
</script>

<template>
  <UDashboardPanel id="fulfillment">
    <template #header>
      <UDashboardNavbar title="Fulfillment Centers - Order Processing">
        <template #leading>
          <UDashboardSidebarCollapse />
        </template>

        <template #right>
          <UButton
            color="neutral"
            variant="outline"
            icon="i-lucide-refresh-cw"
            :loading="loading"
          >
            Refresh
          </UButton>
        </template>
      </UDashboardNavbar>
    </template>

    <template #body>
      <div class="flex h-full min-h-0 flex-col">
        <div class="mb-4 grid gap-4 sm:grid-cols-4">
          <UCard>
            <div class="flex items-center justify-between">
              <div>
                <p class="text-sm text-muted">Total Orders</p>
                <p class="text-2xl font-semibold">{{ totalOrders }}</p>
              </div>
              <UIcon name="i-lucide-package" class="size-8 text-primary" />
            </div>
          </UCard>

          <UCard>
            <div class="flex items-center justify-between">
              <div>
                <p class="text-sm text-muted">Signed Retainers</p>
                <p class="text-2xl font-semibold">{{ signedCount }}</p>
              </div>
              <UIcon name="i-lucide-check-circle" class="size-8 text-success" />
            </div>
          </UCard>

          <UCard>
            <div class="flex items-center justify-between">
              <div>
                <p class="text-sm text-muted">Dropped</p>
                <p class="text-2xl font-semibold">{{ droppedCount }}</p>
              </div>
              <UIcon name="i-lucide-x-circle" class="size-8 text-error" />
            </div>
          </UCard>

          <UCard>
            <div class="flex items-center justify-between">
              <div>
                <p class="text-sm text-muted">Returned Back</p>
                <p class="text-2xl font-semibold">{{ returnedCount }}</p>
              </div>
              <UIcon name="i-lucide-arrow-left-circle" class="size-8 text-warning" />
            </div>
          </UCard>
        </div>

        <div class="flex flex-wrap items-center justify-between gap-3">
          <div class="flex flex-wrap items-center gap-3">
            <UInput
              v-model="query"
              class="max-w-md"
              icon="i-lucide-search"
              placeholder="Search orders..."
            />

            <USelect
              v-model="selectedStage"
              :items="[{ label: 'All Stages', value: 'all' }, ...STAGES.map(s => ({ label: s.label, value: s.key }))]"
              class="w-56"
              value-key="value"
              label-key="label"
            />
          </div>

          <UBadge variant="subtle" :label="`${filteredOrders.length} orders`" />
        </div>

        <div class="mt-4 min-h-0 flex-1 overflow-auto">
          <div class="flex min-h-0 gap-3 pr-2" style="min-width: 1400px;">
            <div
              v-for="stage in STAGES"
              :key="stage.key"
              class="flex min-h-[560px] w-[28rem] flex-col rounded-lg border border-default bg-elevated/20"
            >
              <div class="flex items-center justify-between border-b border-default px-3 py-2">
                <div class="text-sm font-semibold">{{ stage.label }}</div>
                <UBadge
                  variant="subtle"
                  :label="String(ordersByStage.get(stage.key)?.length ?? 0)"
                />
              </div>

              <div class="flex-1 space-y-2 p-2">
                <UCard
                  v-for="order in (ordersByStage.get(stage.key) ?? [])"
                  :key="order.id"
                  class="w-full"
                  :ui="{ body: '!p-2 sm:!p-2' }"
                >
                  <div class="flex items-start justify-between gap-2">
                    <div class="min-w-0">
                      <div class="truncate text-sm font-semibold">{{ order.clientName }}</div>
                      <div class="mt-0.5 text-xs text-muted">{{ order.id }} · {{ order.phone }}</div>
                    </div>
                    <div class="shrink-0 text-sm font-semibold">{{ formatMoney(order.caseValue) }}</div>
                  </div>

                  <div class="mt-2 flex items-center justify-between gap-2">
                    <UBadge variant="subtle" :label="order.state" size="xs" />
                    <div class="text-xs text-muted">{{ order.date }}</div>
                  </div>

                  <div v-if="order.reason" class="mt-2 rounded bg-muted/30 px-2 py-1 text-xs text-muted">
                    {{ order.reason }}
                  </div>

                  <div v-if="order.signedDate" class="mt-2 flex items-center gap-1 text-xs text-success">
                    <UIcon name="i-lucide-check" class="size-3" />
                    <span>Signed: {{ order.signedDate }}</span>
                  </div>
                </UCard>

                <div
                  v-if="(ordersByStage.get(stage.key)?.length ?? 0) === 0"
                  class="rounded-md border border-dashed border-default px-3 py-6 text-center text-xs text-muted"
                >
                  No orders
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </template>
  </UDashboardPanel>
</template>
