<template>
  <table
    role="grid"
    :aria-label="t('el.datepicker.monthTablePrompt')"
    :class="ns.b()"
    @click="handleSeasonTableClick"
    @mousemove="handleMouseMove"
  >
    <tbody ref="tbodyRef">
      <tr>
        <td
          v-for="(cell, key_) in rows"
          :key="key_"
          :ref="(el) => isSelectedCell(cell) && (currentCellRef = el as HTMLElement)"
          :class="getCellStyle(cell)"
          :aria-selected="`${isSelectedCell(cell)}`"
          :aria-label="t(`el.datepicker.month${+cell.text + 1}`)"
          :tabindex="isSelectedCell(cell) ? 0 : -1"
        >
          <div v-if="cell.text < 4">
            <!-- t('el.datepicker.months.' + Seasons[cell.text]) -->
            <span class="cell">
              {{ seasonMap[Seasons[cell.text]] }}
            </span>
          </div>
        </td>
      </tr>
    </tbody>
  </table>
</template>

<script lang="ts" setup>
import { computed, nextTick, ref, watch } from 'vue'
import dayjs from 'dayjs'
import { useLocale, useNamespace } from '@element-plus/hooks'
import { rangeArr } from '@element-plus/components/time-picker'
import { castArray, hasClass } from '@element-plus/utils'
import { basicMonthTableProps } from '../props/basic-season-table'

type MonthCell = {
  column: number
  row: number
  disabled: boolean
  start: boolean
  end: boolean
  text: number
  type: 'normal' | 'today'
  inRange: boolean
}

const datesInMonth = (year: number, month: number, lang: string) => {
  const firstDay = dayjs().locale(lang).startOf('month').month(month).year(year)
  const numOfDays = firstDay.daysInMonth()
  return rangeArr(numOfDays).map((n) => firstDay.add(n, 'day').toDate())
}

const props = defineProps(basicMonthTableProps)
const emit = defineEmits(['changerange', 'pick', 'select'])

const ns = useNamespace('month-table')

const { t, lang } = useLocale()
const tbodyRef = ref<HTMLElement>()
const currentCellRef = ref<HTMLElement>()

// type Seasons = []

const Seasons = ref(['quarter1', 'quarter2', 'quarter3', 'quarter4'])
const seasonMap: any = {
  quarter1: '第一季度',
  quarter2: '第二季度',
  quarter3: '第三季度',
  quarter4: '第四季度',
}
// console.log(props, Seasons, 'Seasons')
const tableRows = ref<MonthCell[]>([])
const lastRow = ref<number>()
const lastColumn = ref<number>()
const rows = computed<MonthCell[]>(() => {
  const rows = tableRows.value
  // console.log(rows, 'rows')
  // const now = dayjs().locale(lang.value).startOf('month')

  for (let i = 0; i < 4; i++) {
    const cell = (rows[i] ||= {
      row: 0,
      column: i,
      type: 'normal',
      inRange: false,
      start: false,
      end: false,
      text: -1,
      disabled: false,
    })
    cell.type = 'normal'
    const calTime = props.date.startOf('year').month(i)

    cell.text = i
    cell.disabled = props.disabledDate?.(calTime.toDate()) || false
  }
  return rows
})

const focus = () => {
  currentCellRef.value?.focus()
}

const getCellStyle = (cell: MonthCell) => {
  const style = {} as any
  const year = props.date.year()
  const today = new Date()
  const month = cell.text

  style.disabled = props.disabledDate
    ? datesInMonth(year, month, lang.value).every(props.disabledDate)
    : false
  style.current =
    castArray(props.parsedValue).findIndex(
      (date) =>
        dayjs.isDayjs(date) && date.year() === year && date.month() === month
    ) >= 0
  // 判断当前月对应的季度
  const curMonth = today.getMonth()
  const isToday = () => {
    return (
      ([0, 1, 2].includes(curMonth) && month === 0) ||
      ([3, 4, 5].includes(curMonth) && month === 1) ||
      ([6, 7, 8].includes(curMonth) && month === 2) ||
      ([9, 10, 11].includes(curMonth) && month === 3)
    )
  }
  style.today = today.getFullYear() === year && isToday()

  return style
}

const isSelectedCell = (cell: MonthCell) => {
  const year = props.date.year()
  const month = cell.text
  return (
    castArray(props.date).findIndex(
      (date) => date.year() === year && date.month() === month
    ) >= 0
  )
}

const handleMouseMove = (event: MouseEvent) => {
  if (!props.rangeState.selecting) return

  let target = event.target as HTMLElement
  if (target.tagName === 'A') {
    target = target.parentNode?.parentNode as HTMLElement
  }
  if (target.tagName === 'DIV') {
    target = target.parentNode as HTMLElement
  }
  if (target.tagName !== 'TD') return

  const row = (target.parentNode as HTMLTableRowElement).rowIndex
  const column = (target as HTMLTableCellElement).cellIndex
  // can not select disabled date
  if (rows.value[column].disabled) return

  // only update rangeState when mouse moves to a new cell
  // this avoids frequent Date object creation and improves performance
  if (row !== lastRow.value || column !== lastColumn.value) {
    lastRow.value = row
    lastColumn.value = column
    emit('changerange', {
      selecting: true,
      endDate: props.date.startOf('year').month(row * 4 + column),
    })
  }
}
const handleSeasonTableClick = (event: MouseEvent | KeyboardEvent) => {
  const target = (event.target as HTMLElement)?.closest(
    'td'
  ) as HTMLTableCellElement
  if (target?.tagName !== 'TD') return
  if (hasClass(target, 'disabled')) return
  const column = target.cellIndex
  const row = (target.parentNode as HTMLTableRowElement).rowIndex
  const season = row * 4 + column
  console.log(season, '...')
  // const newDate = props.date.startOf('year').month(month)
  emit('pick', season)
}

watch(
  () => props.date,
  async () => {
    if (tbodyRef.value?.contains(document.activeElement)) {
      await nextTick()
      currentCellRef.value?.focus()
    }
  }
)

defineExpose({
  /**
   * @description focus current cell
   */
  focus,
})
</script>
