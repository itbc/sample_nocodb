<script setup lang="ts">
import type { SelectProps } from 'ant-design-vue'
import type { ColumnType, LinkToAnotherRecordType } from 'nocodb-sdk'
import { RelationTypes, UITypes, isSystemColumn, isVirtualCol } from 'nocodb-sdk'
import { ActiveViewInj, MetaInj, computed, inject, ref, resolveComponent, useViewColumns } from '#imports'

const { modelValue, isSort } = defineProps<{
  modelValue?: string
  isSort?: boolean
}>()

const emit = defineEmits(['update:modelValue'])

const meta = inject(MetaInj, ref())

const localValue = computed({
  get: () => modelValue,
  set: (val) => emit('update:modelValue', val),
})

const activeView = inject(ActiveViewInj, ref())

const { showSystemFields, metaColumnById } = useViewColumns(activeView, meta)

const options = computed<SelectProps['options']>(() =>
  meta.value?.columns
    ?.filter((c: ColumnType) => {
      if (isSystemColumn(metaColumnById?.value?.[c.id!])) {
        return (
          /** if the field is used in filter, then show it anyway */
          localValue.value === c.id ||
          /** hide system columns if not enabled */
          showSystemFields.value
        )
      } else if (c.uidt === UITypes.QrCode || c.uidt === UITypes.Barcode || c.uidt === UITypes.ID) {
        return false
      } else if (isSort) {
        /** ignore hasmany and manytomany relations if it's using within sort menu */
        return !(
          c.uidt === UITypes.LinkToAnotherRecord && (c.colOptions as LinkToAnotherRecordType).type !== RelationTypes.BELONGS_TO
        )
        /** ignore virtual fields which are system fields ( mm relation ) and qr code fields */
      } else {
        const isVirtualSystemField = c.colOptions && c.system
        return !isVirtualSystemField
      }
    })
    .map((c: ColumnType) => ({
      value: c.id,
      label: c.title,
      icon: h(
        isVirtualCol(c) ? resolveComponent('SmartsheetHeaderVirtualCellIcon') : resolveComponent('SmartsheetHeaderCellIcon'),
        {
          columnMeta: c,
        },
      ),
      c,
    })),
)

const filterOption = (input: string, option: any) => option.label.toLowerCase()?.includes(input.toLowerCase())

// when a new filter is created, select a field by default
if (!localValue.value) {
  localValue.value = (options.value?.[0].value as string) || ''
}
</script>

<template>
  <a-select
    v-model:value="localValue"
    :dropdown-match-select-width="false"
    show-search
    :placeholder="$t('placeholder.selectField')"
    :filter-option="filterOption"
    dropdown-class-name="nc-dropdown-toolbar-field-list"
  >
    <a-select-option v-for="option in options" :key="option.value" :label="option.label" :value="option.value">
      <div class="flex gap-2 items-center items-center h-full">
        <component :is="option.icon" class="min-w-5 !mx-0" />

        <span class="min-w-0"> {{ option.label }}</span>
      </div>
    </a-select-option>
  </a-select>
</template>

<style lang="scss">
.ant-select-selection-search-input {
  box-shadow: none !important;
}
::-webkit-scrollbar {
  -webkit-appearance: none;
  width: 7px;
}

::-webkit-scrollbar-thumb {
  border-radius: 4px;
  background-color: rgba(0, 0, 0, 0.5);
  box-shadow: 0 0 1px rgba(255, 255, 255, 0.5);
}
</style>
