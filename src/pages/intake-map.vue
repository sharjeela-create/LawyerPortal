<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'
import usSvgFallbackRaw from '../assets/us.svg?raw'

type StateStatus = 'active' | 'low' | 'inactive'

type StateIntake = {
  code: string
  name: string
  currentVolume: number
  targetVolume: number
  salesNext30Days: number
  status: StateStatus
  fulfilled: number
  pending: number
}

const loading = ref(false)
const tooltip = ref({ open: false, x: 0, y: 0, state: null as StateIntake | null })
const mapRoot = ref<HTMLDivElement | null>(null)
const tooltipEl = ref<HTMLDivElement | null>(null)
const selectedStatus = ref<'all' | StateStatus>('all')

type IntakeOrder = {
  id: string
  stateCode: string
  volume: number
  salesForecast: number
  days: number
  createdAt: string
}

const createOrderOpen = ref(false)
const orderForm = ref({
  stateCode: '' as string,
  volume: 1 as number,
  salesForecast: 5 as number,
  days: 30 as number
})

const resetOrderForm = () => {
  orderForm.value = {
    stateCode: '',
    volume: 1,
    salesForecast: 5,
    days: 30
  }
}

const orders = ref<IntakeOrder[]>([])

const openCreateOrder = () => {
  createOrderOpen.value = true
}

const handleCreateOrderOpenUpdate = (v: boolean) => {
  createOrderOpen.value = v
}

const mockStates = ref<StateIntake[]>([
  { code: 'CA', name: 'California', currentVolume: 45, targetVolume: 50, salesNext30Days: 5, status: 'active', fulfilled: 40, pending: 5 },
  { code: 'TX', name: 'Texas', currentVolume: 38, targetVolume: 40, salesNext30Days: 4, status: 'active', fulfilled: 35, pending: 3 },
  { code: 'FL', name: 'Florida', currentVolume: 22, targetVolume: 30, salesNext30Days: 3, status: 'low', fulfilled: 20, pending: 2 },
  { code: 'NY', name: 'New York', currentVolume: 35, targetVolume: 40, salesNext30Days: 5, status: 'active', fulfilled: 32, pending: 3 },
  { code: 'PA', name: 'Pennsylvania', currentVolume: 18, targetVolume: 25, salesNext30Days: 2, status: 'low', fulfilled: 16, pending: 2 },
  { code: 'IL', name: 'Illinois', currentVolume: 28, targetVolume: 30, salesNext30Days: 3, status: 'active', fulfilled: 25, pending: 3 },
  { code: 'OH', name: 'Ohio', currentVolume: 15, targetVolume: 20, salesNext30Days: 2, status: 'low', fulfilled: 13, pending: 2 },
  { code: 'GA', name: 'Georgia', currentVolume: 20, targetVolume: 25, salesNext30Days: 3, status: 'low', fulfilled: 18, pending: 2 },
  { code: 'NC', name: 'North Carolina', currentVolume: 12, targetVolume: 15, salesNext30Days: 1, status: 'low', fulfilled: 11, pending: 1 },
  { code: 'MI', name: 'Michigan', currentVolume: 8, targetVolume: 15, salesNext30Days: 1, status: 'low', fulfilled: 7, pending: 1 },
  { code: 'WA', name: 'Washington', currentVolume: 25, targetVolume: 30, salesNext30Days: 4, status: 'active', fulfilled: 22, pending: 3 },
  { code: 'AZ', name: 'Arizona', currentVolume: 16, targetVolume: 20, salesNext30Days: 2, status: 'low', fulfilled: 14, pending: 2 },
  { code: 'MA', name: 'Massachusetts', currentVolume: 19, targetVolume: 25, salesNext30Days: 3, status: 'low', fulfilled: 17, pending: 2 },
  { code: 'TN', name: 'Tennessee', currentVolume: 10, targetVolume: 15, salesNext30Days: 1, status: 'low', fulfilled: 9, pending: 1 },
  { code: 'CO', name: 'Colorado', currentVolume: 14, targetVolume: 20, salesNext30Days: 2, status: 'low', fulfilled: 12, pending: 2 },
  { code: 'OR', name: 'Oregon', currentVolume: 11, targetVolume: 15, salesNext30Days: 1, status: 'low', fulfilled: 10, pending: 1 },
  { code: 'NV', name: 'Nevada', currentVolume: 9, targetVolume: 12, salesNext30Days: 1, status: 'low', fulfilled: 8, pending: 1 },
  { code: 'VA', name: 'Virginia', currentVolume: 13, targetVolume: 18, salesNext30Days: 2, status: 'low', fulfilled: 11, pending: 2 },
  { code: 'MN', name: 'Minnesota', currentVolume: 7, targetVolume: 12, salesNext30Days: 1, status: 'low', fulfilled: 6, pending: 1 },
  { code: 'WI', name: 'Wisconsin', currentVolume: 6, targetVolume: 10, salesNext30Days: 1, status: 'low', fulfilled: 5, pending: 1 }
])

const filteredStates = computed(() => {
  if (selectedStatus.value === 'all') return mockStates.value
  return mockStates.value.filter(s => s.status === selectedStatus.value)
})

const stateByCode = computed(() => {
  const map = new Map<string, StateIntake>()
  mockStates.value.forEach(s => map.set(s.code, s))
  return map
})

const totalVolume = computed(() => mockStates.value.reduce((sum, s) => sum + s.currentVolume, 0))
const totalTarget = computed(() => mockStates.value.reduce((sum, s) => sum + s.targetVolume, 0))
const totalFulfilled = computed(() => mockStates.value.reduce((sum, s) => sum + s.fulfilled, 0))
const totalPending = computed(() => mockStates.value.reduce((sum, s) => sum + s.pending, 0))

const statusOptions = [
  { label: 'All States', value: 'all' },
  { label: 'Active', value: 'active' },
  { label: 'Low Volume', value: 'low' },
  { label: 'Inactive', value: 'inactive' }
]

const orderDaysOptions = [
  { label: 'Next 7 days', value: 7 },
  { label: 'Next 14 days', value: 14 },
  { label: 'Next 30 days', value: 30 },
  { label: 'Next 60 days', value: 60 }
]

const orderStateOptions = computed(() => {
  return mockStates.value
    .slice()
    .sort((a, b) => a.name.localeCompare(b.name))
    .map((s) => ({ label: `${s.name} (${s.code})`, value: s.code }))
})

const createOrder = () => {
  const stateCode = String(orderForm.value.stateCode || '').trim().toUpperCase()
  const volume = Number(orderForm.value.volume)
  const salesForecast = Number(orderForm.value.salesForecast)
  const days = Number(orderForm.value.days)

  if (!stateCode || !Number.isFinite(volume) || volume <= 0) return
  if (!Number.isFinite(salesForecast) || salesForecast < 0) return
  if (!Number.isFinite(days) || days <= 0) return

  orders.value.unshift({
    id: `O-${Math.random().toString(16).slice(2, 10).toUpperCase()}`,
    stateCode,
    volume: Math.round(volume),
    salesForecast: Math.round(salesForecast),
    days: Math.round(days),
    createdAt: new Date().toISOString()
  })

  const idx = mockStates.value.findIndex((s) => s.code === stateCode)
  if (idx !== -1) {
    const s = mockStates.value[idx]
    s.currentVolume += Math.round(volume)
    s.pending += Math.round(volume)
    if (days === 30) s.salesNext30Days += Math.round(salesForecast)
    s.status = s.currentVolume >= s.targetVolume ? 'active' : 'low'
  }

  createOrderOpen.value = false
  resetOrderForm()
  applyMapColors()
}

const getStatusColor = (status: StateStatus) => {
  if (status === 'active') return '#22c55e'
  if (status === 'low') return '#eab308'
  return '#ef4444'
}

const getStatusLabel = (status: StateStatus) => {
  if (status === 'active') return 'Active'
  if (status === 'low') return 'Low Volume'
  return 'Inactive'
}

const US_SVG_URL = 'https://simplemaps.com/static/demos/resources/svg-library/svgs/us.svg'
const US_SVG_CACHE_KEY = 'lawyer-us-map-svg-cache-v1'

const applyMapColors = () => {
  const root = mapRoot.value
  if (!root) return
  const svg = root.querySelector('svg')
  if (!svg) return

  const visibleSet = new Set(filteredStates.value.map(s => s.code))
  const paths = svg.querySelectorAll('path[data-id]')
  paths.forEach((p) => {
    const code = p.getAttribute('data-id') || p.getAttribute('id')
    if (!code) return
    const state = stateByCode.value.get(code)
    const isVisible = selectedStatus.value === 'all' ? true : visibleSet.has(code)

    const fill = state && isVisible ? getStatusColor(state.status) : '#e5e7eb'
    const stroke = '#0b0b0b'

    p.style.setProperty('fill', fill, 'important')
    p.style.setProperty('stroke', stroke, 'important')
    p.style.setProperty('stroke-width', '0.8', 'important')
    p.style.cursor = state ? 'pointer' : 'default'
    p.style.opacity = isVisible ? '1' : '0.15'
  })

  applyStateLabels()
}

const applyStateLabels = () => {
  const root = mapRoot.value
  if (!root) return
  const svg = root.querySelector('svg') as SVGSVGElement | null
  if (!svg) return

  const old = svg.querySelector('#state-labels')
  if (old) old.remove()

  const g = document.createElementNS('http://www.w3.org/2000/svg', 'g')
  g.setAttribute('id', 'state-labels')
  g.setAttribute('pointer-events', 'none')

  const paths = svg.querySelectorAll('path[data-id]')
  paths.forEach((p) => {
    const code = p.getAttribute('data-id') || p.getAttribute('id')
    if (!code) return

    let bbox: DOMRect
    try {
      bbox = (p as unknown as SVGGraphicsElement).getBBox()
    } catch {
      return
    }

    const cx = bbox.x + bbox.width / 2
    const cy = bbox.y + bbox.height / 2

    const state = stateByCode.value.get(code)
    const isVisible = selectedStatus.value === 'all' ? true : filteredStates.value.some(s => s.code === code)
    if (!isVisible) return

    const fontSize = Math.max(6, Math.min(bbox.width, bbox.height) / 3)
    const fill = '#ffffff'

    const text = document.createElementNS('http://www.w3.org/2000/svg', 'text')
    text.textContent = code
    text.setAttribute('x', String(cx))
    text.setAttribute('y', String(cy))
    text.setAttribute('text-anchor', 'middle')
    text.setAttribute('dominant-baseline', 'middle')
    text.style.setProperty('font-size', `${fontSize}px`, 'important')
    text.style.setProperty('font-weight', '700', 'important')
    text.style.setProperty('fill', fill, 'important')
    text.style.setProperty('paint-order', 'stroke', 'important')
    text.style.setProperty('stroke', 'rgba(0,0,0,0.5)', 'important')
    text.style.setProperty('stroke-width', '1', 'important')

    g.appendChild(text)
  })

  svg.appendChild(g)
}

const handleStateEnter = (evt: Event) => {
  const target = evt.target as HTMLElement | null
  if (!target) return
  const code = target.getAttribute('data-id') || target.getAttribute('id')
  if (!code) return
  const state = stateByCode.value.get(code) ?? null
  if (!state) return

  tooltip.value.state = state
  tooltip.value.open = true
}

const handleStateLeave = () => {
  tooltip.value.open = false
  tooltip.value.state = null
}

const handleMouseMove = (evt: MouseEvent) => {
  if (!tooltip.value.open) return

  const root = mapRoot.value
  if (!root) return
  const rect = root.getBoundingClientRect()

  const offset = 6
  const rawX = evt.clientX - rect.left + offset
  const rawY = evt.clientY - rect.top + offset

  const w = tooltipEl.value?.offsetWidth ?? 0
  const h = tooltipEl.value?.offsetHeight ?? 0

  const maxX = Math.max(0, rect.width - w - 4)
  const maxY = Math.max(0, rect.height - h - 4)

  tooltip.value.x = Math.max(4, Math.min(rawX, maxX))
  tooltip.value.y = Math.max(4, Math.min(rawY, maxY))
}

const bindSvgEvents = () => {
  const root = mapRoot.value
  if (!root) return
  const svg = root.querySelector('svg')
  if (!svg) return

  const paths = svg.querySelectorAll('path[data-id]')
  paths.forEach((p) => {
    p.addEventListener('mouseenter', handleStateEnter)
    p.addEventListener('mouseleave', handleStateLeave)
  })

  svg.addEventListener('mousemove', handleMouseMove)
}

const unbindSvgEvents = () => {
  const root = mapRoot.value
  if (!root) return
  const svg = root.querySelector('svg')
  if (!svg) return

  const paths = svg.querySelectorAll('path[data-id]')
  paths.forEach((p) => {
    p.removeEventListener('mouseenter', handleStateEnter)
    p.removeEventListener('mouseleave', handleStateLeave)
  })

  svg.removeEventListener('mousemove', handleMouseMove)
}

onMounted(() => {
  const run = async () => {
    if (!mapRoot.value) return

    try {
      const res = await fetch(US_SVG_URL)
      if (!res.ok) throw new Error('Failed to load map')
      const svgText = await res.text()

      try {
        localStorage.setItem(US_SVG_CACHE_KEY, svgText)
      } catch {
        // ignore
      }

      mapRoot.value.innerHTML = svgText
      bindSvgEvents()
      applyMapColors()
    } catch {
      const cached = (() => {
        try {
          return localStorage.getItem(US_SVG_CACHE_KEY)
        } catch {
          return null
        }
      })()

      mapRoot.value.innerHTML = cached || usSvgFallbackRaw
      bindSvgEvents()
      applyMapColors()
    }
  }

  void run()
})

onUnmounted(() => {
  unbindSvgEvents()
})

watch([selectedStatus, filteredStates], () => {
  applyMapColors()
})

const columns = [
  { accessorKey: 'name', header: 'State' },
  { accessorKey: 'currentVolume', header: 'Current Volume' },
  { accessorKey: 'targetVolume', header: 'Target Volume' },
  { accessorKey: 'salesNext30Days', header: 'Sales (Next 30 Days)' },
  { accessorKey: 'fulfilled', header: 'Fulfilled' },
  { accessorKey: 'pending', header: 'Pending' },
  { accessorKey: 'status', header: 'Status' }
]
</script>

<template>
  <UDashboardPanel id="intake-map">
    <template #header>
      <UDashboardNavbar title="Intake Map - State Volume Tracking">
        <template #leading>
          <UDashboardSidebarCollapse />
        </template>

        <template #right>
          <UButton
            color="primary"
            variant="solid"
            icon="i-lucide-plus"
            @click="openCreateOrder"
          >
            Create Order
          </UButton>

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
      <div class="space-y-4">
        <UModal
          v-if="createOrderOpen"
          :open="true"
          title="Create Intake Order"
          :dismissible="false"
          @update:open="handleCreateOrderOpenUpdate"
        >
          <template #body="{ close }">
            <div class="space-y-4">
              <div class="text-sm text-muted">Choose a state, volume, and forecast window.</div>

              <UFormField label="State" required>
                <USelect
                  v-model="orderForm.stateCode"
                  :items="orderStateOptions"
                  value-key="value"
                  label-key="label"
                  placeholder="Select a state"
                />
              </UFormField>

              <UFormField label="Volume" description="How many orders do you want?" required>
                <UInput v-model.number="orderForm.volume" type="number" min="1" />
              </UFormField>

              <UFormField label="Sales Forecast" description="Expected sales in the selected window">
                <UInput v-model.number="orderForm.salesForecast" type="number" min="0" />
              </UFormField>

              <UFormField label="Days Window" description="Forecast window for the sales estimate" required>
                <USelect
                  v-model="orderForm.days"
                  :items="orderDaysOptions"
                  value-key="value"
                  label-key="label"
                />
              </UFormField>

              <div class="flex items-center justify-end gap-2">
                <UButton
                  color="neutral"
                  variant="outline"
                  @click="() => { resetOrderForm(); close() }"
                >
                  Cancel
                </UButton>
                <UButton
                  color="primary"
                  variant="solid"
                  :disabled="!String(orderForm.stateCode || '').trim() || Number(orderForm.volume) <= 0"
                  @click="() => { createOrder(); close() }"
                >
                  Create
                </UButton>
              </div>
            </div>
          </template>
        </UModal>

        <div class="grid gap-4 sm:grid-cols-4">
          <UCard>
            <div class="flex items-center justify-between">
              <div>
                <p class="text-sm text-muted">Total Volume</p>
                <p class="text-2xl font-semibold">{{ totalVolume }}</p>
              </div>
              <UIcon name="i-lucide-trending-up" class="size-8 text-primary" />
            </div>
          </UCard>

          <UCard>
            <div class="flex items-center justify-between">
              <div>
                <p class="text-sm text-muted">Target Volume</p>
                <p class="text-2xl font-semibold">{{ totalTarget }}</p>
              </div>
              <UIcon name="i-lucide-target" class="size-8 text-primary" />
            </div>
          </UCard>

          <UCard>
            <div class="flex items-center justify-between">
              <div>
                <p class="text-sm text-muted">Fulfilled</p>
                <p class="text-2xl font-semibold">{{ totalFulfilled }}</p>
              </div>
              <UIcon name="i-lucide-check-circle" class="size-8 text-success" />
            </div>
          </UCard>

          <UCard>
            <div class="flex items-center justify-between">
              <div>
                <p class="text-sm text-muted">Pending</p>
                <p class="text-2xl font-semibold">{{ totalPending }}</p>
              </div>
              <UIcon name="i-lucide-clock" class="size-8 text-warning" />
            </div>
          </UCard>
        </div>

        <div>
          <UCard>
            <div class="space-y-3">
              <h3 class="font-semibold">Status Legend</h3>
              <div class="grid gap-3 sm:grid-cols-3">
                <div class="flex items-center gap-2">
                  <div class="size-4 rounded-full bg-green-500" />
                  <span class="text-sm">Active - Meeting Target</span>
                </div>
                <div class="flex items-center gap-2">
                  <div class="size-4 rounded-full bg-yellow-500" />
                  <span class="text-sm">Low Volume - Below Target</span>
                </div>
                <div class="flex items-center gap-2">
                  <div class="size-4 rounded-full bg-red-500" />
                  <span class="text-sm">Inactive - No Volume</span>
                </div>
              </div>
            </div>
          </UCard>
        </div>

        <div class="flex items-center justify-between">
          <USelect
            v-model="selectedStatus"
            class="w-64"
            :items="statusOptions"
            value-key="value"
            label-key="label"
          />

          <UBadge variant="subtle" :label="`${filteredStates.length} states`" />
        </div>

        <UCard :ui="{ body: 'p-4' }">
          <div class="relative">
            <div
              ref="mapRoot"
              class="w-full overflow-auto rounded-lg bg-white"
              style="max-height: 520px;"
            />

            <div
              v-if="tooltip.open && tooltip.state"
              ref="tooltipEl"
              class="pointer-events-none absolute z-10 rounded-lg border border-default bg-elevated px-3 py-2 shadow-lg"
              :style="{ left: `${tooltip.x}px`, top: `${tooltip.y}px` }"
            >
              <div class="font-semibold">{{ tooltip.state.name }} ({{ tooltip.state.code }})</div>
              <div class="mt-1 flex items-center gap-2">
                <UBadge
                  :color="tooltip.state.status === 'active' ? 'success' : tooltip.state.status === 'low' ? 'warning' : 'error'"
                  variant="subtle"
                  :label="getStatusLabel(tooltip.state.status)"
                  size="xs"
                />
              </div>
              <div class="mt-2 space-y-1 text-xs text-muted">
                <div>Current: {{ tooltip.state.currentVolume }} / Target: {{ tooltip.state.targetVolume }}</div>
                <div>Sales (30d): {{ tooltip.state.salesNext30Days }}</div>
                <div>Fulfilled: {{ tooltip.state.fulfilled }} | Pending: {{ tooltip.state.pending }}</div>
              </div>
            </div>
          </div>
        </UCard>

        <UCard :ui="{ body: 'p-0' }">
          <UTable
            :data="filteredStates"
            :columns="columns"
            :ui="{
              base: 'w-full',
              thead: '[&>tr]:bg-elevated/50',
              tbody: '[&>tr]:hover:bg-muted/50',
              th: 'px-4 py-3 text-left',
              td: 'px-4 py-3 align-top'
            }"
          >
            <template #status-cell="{ row }">
              <UBadge
                :color="row.original.status === 'active' ? 'success' : row.original.status === 'low' ? 'warning' : 'error'"
                variant="subtle"
                :label="getStatusLabel(row.original.status)"
                size="xs"
              />
            </template>

            <template #currentVolume-cell="{ row }">
              <div class="font-semibold">{{ row.original.currentVolume }}</div>
            </template>

            <template #targetVolume-cell="{ row }">
              <div class="font-semibold">{{ row.original.targetVolume }}</div>
            </template>
          </UTable>
        </UCard>
      </div>
    </template>
  </UDashboardPanel>
</template>
