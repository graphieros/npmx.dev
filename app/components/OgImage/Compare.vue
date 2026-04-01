<script setup lang="ts">
import { computed, ref } from 'vue'
import { encodePackageName } from '#shared/utils/npm'

const props = withDefaults(
  defineProps<{
    packages?: string | string[]
    emptyDescription?: string
    primaryColor?: string
  }>(),
  {
    packages: () => [],
    emptyDescription: 'Compare npm packages side-by-side',
    primaryColor: '#60a5fa',
  },
)

// OG image is 1200×600. With 80px horizontal padding, content width is 1040px.
const OG_PADDING_X = 80
const CONTENT_WIDTH = 1200 - OG_PADDING_X * 2

const ACCENT_COLORS = [
  '#60a5fa',
  '#f472b6',
  '#34d399',
  '#fbbf24',
  '#a78bfa',
  '#fb923c',
  '#22d3ee',
  '#e879f9',
  '#4ade80',
  '#f87171',
  '#38bdf8',
  '#facc15',
]

// Tier thresholds
const FULL_MAX = 4
const COMPACT_MAX = 6
const GRID_MAX = 12
const SUMMARY_TOP_COUNT = 3

const displayPackages = computed(() => {
  const raw = props.packages
  return (typeof raw === 'string' ? raw.split(',') : raw).map(p => p.trim()).filter(Boolean)
})

type LayoutTier = 'full' | 'compact' | 'grid' | 'summary'
const layoutTier = computed<LayoutTier>(() => {
  const count = displayPackages.value.length
  if (count <= FULL_MAX) return 'full'
  if (count <= COMPACT_MAX) return 'compact'
  if (count <= GRID_MAX) return 'grid'
  return 'summary'
})

interface PkgStats {
  name: string
  downloads: number
  version: string
  color: string
}

const stats = ref<PkgStats[]>([])

const FETCH_TIMEOUT_MS = 2500

if (layoutTier.value !== 'summary') {
  try {
    const results = await Promise.all(
      displayPackages.value.map(async (name, index) => {
        const encoded = encodePackageName(name)
        const [dlData, pkgData] = await Promise.all([
          $fetch<{ downloads: number }>(
            `https://api.npmjs.org/downloads/point/last-week/${encoded}`,
            { timeout: FETCH_TIMEOUT_MS },
          ).catch(() => null),
          $fetch<{ 'dist-tags'?: { latest?: string } }>(
            `https://registry.npmjs.org/${encoded}`,
            {
              timeout: FETCH_TIMEOUT_MS,
              headers: { Accept: 'application/vnd.npm.install-v1+json' },
            },
          ).catch(() => null),
        ])
        return {
          name,
          downloads: dlData?.downloads ?? 0,
          version: pkgData?.['dist-tags']?.latest ?? '',
          color: ACCENT_COLORS[index % ACCENT_COLORS.length]!,
        }
      }),
    )
    // Sort by downloads descending for readability
    stats.value = results.sort((a, b) => b.downloads - a.downloads)
  } catch {
    stats.value = displayPackages.value.map((name, index) => ({
      name,
      downloads: 0,
      version: '',
      color: ACCENT_COLORS[index % ACCENT_COLORS.length]!,
    }))
  }
}

const maxDownloads = computed(() => Math.max(...stats.value.map(s => s.downloads), 1))

function formatDownloads(n: number): string {
  if (n === 0) return '—'
  return Intl.NumberFormat('en', {
    notation: 'compact',
    maximumFractionDigits: 1,
  }).format(n)
}

const BAR_MIN_PCT = 5
const BAR_MAX_PCT = 100

function barPct(downloads: number): string {
  if (downloads <= 0) return '0%'
  const pct = (downloads / maxDownloads.value) * 100
  return `${Math.min(BAR_MAX_PCT, Math.max(pct, BAR_MIN_PCT))}%`
}

// Grid layout: aim for 2 balanced rows
const GRID_COLS_SMALL = 4 // for up to 8 packages (2 rows of 4)
const GRID_COLS_LARGE = 5 // for 9+ packages (2 rows of 5)

const gridColumns = computed(() => {
  const count = stats.value.length
  if (count <= GRID_COLS_SMALL * 2) return GRID_COLS_SMALL
  return GRID_COLS_LARGE
})

const GRID_ITEM_GAP = 10
const gridItemWidth = computed(
  () => `${Math.floor(CONTENT_WIDTH / gridColumns.value) - GRID_ITEM_GAP}px`,
)

const gridRows = computed(() => {
  const cols = gridColumns.value
  const rows: PkgStats[][] = []
  for (let i = 0; i < stats.value.length; i += cols) {
    rows.push(stats.value.slice(i, i + cols))
  }
  return rows
})

const summaryTopNames = computed(() => displayPackages.value.slice(0, SUMMARY_TOP_COUNT))
const summaryRemainder = computed(() =>
  Math.max(0, displayPackages.value.length - SUMMARY_TOP_COUNT),
)
</script>

<template>
  <div
    class="h-full w-full flex flex-col justify-center relative overflow-hidden bg-[#050505] text-[#fafafa] px-20"
    style="font-family: 'Geist Mono', sans-serif"
  >
    <div class="relative z-10 flex flex-col gap-5">
      <!-- Icon + title row -->
      <div class="flex items-start gap-4">
        <div
          class="flex items-center justify-center w-16 h-16 p-3.5 rounded-xl shadow-lg"
          :style="{ background: `linear-gradient(to top right, #3b82f6, ${primaryColor})` }"
        >
          <svg
            width="36"
            height="36"
            viewBox="0 0 24 24"
            fill="none"
            stroke="white"
            stroke-width="2.5"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <path d="m7.5 4.27 9 5.15" />
            <path
              d="M21 8a2 2 0 0 0-1-1.73l-7-4a2 2 0 0 0-2 0l-7 4A2 2 0 0 0 3 8v8a2 2 0 0 0 1 1.73l7 4a2 2 0 0 0 2 0l7-4A2 2 0 0 0 21 16Z"
            />
            <path d="m3.3 7 8.7 5 8.7-5" />
            <path d="M12 22V12" />
          </svg>
        </div>

        <h1 class="text-7xl font-bold tracking-tight">
          <span
            class="opacity-80 tracking-[-0.1em]"
            :style="{ color: primaryColor }"
            style="margin-right: 0.25rem"
            >./</span
          >compare
        </h1>
      </div>

      <!-- Empty state -->
      <div
        v-if="displayPackages.length === 0"
        class="text-4xl text-[#a3a3a3]"
        style="font-family: 'Geist', sans-serif"
      >
        {{ emptyDescription }}
      </div>

      <!-- FULL layout (1-4 packages): name + downloads + version badge + bar -->
      <div v-else-if="layoutTier === 'full'" class="flex flex-col gap-2">
        <div v-for="pkg in stats" :key="pkg.name" class="flex flex-col gap-1">
          <div class="flex items-center gap-3" style="font-family: 'Geist', sans-serif">
            <span
              class="text-2xl font-semibold tracking-tight truncate max-w-[400px]"
              :style="{ color: pkg.color }"
            >
              {{ pkg.name }}
            </span>
            <span
              v-if="pkg.version"
              class="text-lg px-2 py-0.5 rounded-md border"
              :style="{
                color: pkg.color,
                backgroundColor: pkg.color + '10',
                borderColor: pkg.color + '30',
              }"
            >
              {{ pkg.version }}
            </span>
            <span class="text-3xl font-bold text-[#fafafa]">
              {{ formatDownloads(pkg.downloads) }}/wk
            </span>
          </div>
          <div
            class="h-6 rounded-md"
            :style="{
              width: barPct(pkg.downloads),
              background: `linear-gradient(90deg, ${pkg.color}50, ${pkg.color}20)`,
            }"
          />
        </div>
      </div>

      <!-- COMPACT layout (5-6 packages): name + downloads + thinner bar, no version -->
      <div v-else-if="layoutTier === 'compact'" class="flex flex-col gap-2">
        <div v-for="pkg in stats" :key="pkg.name" class="flex flex-col gap-0.5">
          <div class="flex items-center gap-2" style="font-family: 'Geist', sans-serif">
            <span
              class="text-xl font-semibold tracking-tight truncate max-w-[300px]"
              :style="{ color: pkg.color }"
            >
              {{ pkg.name }}
            </span>
            <span
              v-if="pkg.version"
              class="text-sm px-1.5 py-0.5 rounded border"
              :style="{
                color: pkg.color,
                backgroundColor: pkg.color + '10',
                borderColor: pkg.color + '30',
              }"
            >
              {{ pkg.version }}
            </span>
            <span class="text-xl font-bold text-[#fafafa]">
              {{ formatDownloads(pkg.downloads) }}/wk
            </span>
          </div>
          <div
            class="h-3 rounded-sm"
            :style="{
              width: barPct(pkg.downloads),
              background: `linear-gradient(90deg, ${pkg.color}50, ${pkg.color}20)`,
            }"
          />
        </div>
      </div>

      <!-- GRID layout (7-12 packages): packages in a side-by-side grid -->
      <div
        v-else-if="layoutTier === 'grid'"
        class="flex flex-col gap-6"
        style="font-family: 'Geist', sans-serif"
      >
        <div v-for="(row, ri) in gridRows" :key="ri" class="flex items-start">
          <!-- Using <span> as grid items because Satori treats <div> as full-width flex columns -->
          <span
            v-for="pkg in row"
            :key="pkg.name"
            :style="{
              display: 'flex',
              flexDirection: 'column',
              gap: '2px',
              width: gridItemWidth,
            }"
          >
            <span class="flex items-baseline gap-1.5">
              <span
                class="font-semibold tracking-tight"
                :style="{
                  fontSize: '18px',
                  overflow: 'hidden',
                  textOverflow: 'ellipsis',
                  whiteSpace: 'nowrap',
                  color: pkg.color,
                }"
                >{{ pkg.name }}</span
              >
              <span
                v-if="pkg.version"
                class="text-xs text-[#525252]"
              >{{ pkg.version }}</span>
            </span>
            <span class="flex items-baseline gap-0.5">
              <span class="text-2xl font-bold text-[#e5e5e5]">{{
                formatDownloads(pkg.downloads)
              }}</span>
              <span class="text-sm font-medium text-[#737373]">/wk</span>
            </span>
          </span>
        </div>
      </div>

      <!-- SUMMARY layout (13+ packages): top names + remainder count -->
      <div v-else class="flex flex-col gap-4" style="font-family: 'Geist', sans-serif">
        <div class="text-5xl font-bold tracking-tight">
          Comparing {{ displayPackages.length }} packages
        </div>
        <div class="flex items-center gap-2 text-2xl text-[#a3a3a3]">
          <span
            v-for="(name, i) in summaryTopNames"
            :key="name"
            class="font-semibold"
            :style="{ color: ACCENT_COLORS[i % ACCENT_COLORS.length] }"
          >
            {{ name }}<span v-if="i < summaryTopNames.length - 1" class="text-[#525252]">,</span>
          </span>
          <span v-if="summaryRemainder > 0" class="text-[#737373]">
            +{{ summaryRemainder }} more
          </span>
        </div>
      </div>
    </div>

    <!-- Branding -->
    <div
      class="absolute bottom-6 right-20 text-lg font-semibold tracking-tight text-[#525252]"
      style="font-family: 'Geist Mono', sans-serif"
    >
      <span :style="{ color: primaryColor }" class="opacity-80 tracking-[-0.1em]">./</span>npmx
    </div>

    <div
      class="absolute -top-32 -inset-ie-32 w-[550px] h-[550px] rounded-full blur-3xl"
      :style="{ backgroundColor: primaryColor + '10' }"
    />
  </div>
</template>
