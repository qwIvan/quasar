<template>
  <div class="q-data-table">
    <template v-if="hasToolbar && toolbar === ''">
      <div class="q-data-table-toolbar upper-toolbar row reverse-wrap items-center justify-end">
        <div v-if="config.title" class="q-data-table-title ellipsis auto" v-html="config.title"></div>
        <div class="row items-end">
          <button v-if="config.refresh && !refreshing" class="primary clear" @click="refresh">
            <i>refresh</i>
          </button>
          <button v-if="refreshing" class="disabled">
            <i class="animate-spin-reverse">cached</i>
          </button>
          <q-select
            type="toggle"
            v-if="config.columnPicker"
            v-model="columnSelection"
            :options="columnSelectionOptions"
            :static-label="labels.columns"
            class="text-right"
            style="margin-left: 10px"
          ></q-select>
        </div>
      </div>
    </template>

    <div class="q-data-table-toolbar upper-toolbar row reverse-wrap items-center justify-end q-data-table-selection" v-show="toolbar === 'selection'">
      <div class="auto">
        {{ rowsSelected }}
        <span v-if="rowsSelected === 1">{{ labels.selected.singular }}</span>
        <span v-else>{{ labels.selected.plural }}</span>
        <button class="primary clear small" @click="clearSelection()">{{ labels.clear }}</button>
      </div>
      <div>
        <slot name="selection" :rows="selectedRows"></slot>
      </div>
    </div>

    <table-filter v-if="filteringCols.length" :filtering="filtering" :columns="filteringCols" :labels="labels"></table-filter>

    <template v-if="responsive">
      <div v-if="message" class="q-data-table-message row items-center justify-center" v-html="message"></div>
      <div v-else :style="bodyStyle" style="overflow: auto">
        <table class="q-table horizontal-delimiter responsive" ref="body">
          <tbody>
            <tr v-for="(row, index) in rows">
              <td v-if="config.selection">
                <q-checkbox v-if="config.selection === 'multiple'" v-model="rowSelection[index]"></q-checkbox>
                <q-radio v-else v-model="rowSelection[0]" :val="index"></q-radio>
              </td>
              <td v-for="col in cols" :data-th="col.label" :style="formatStyle(col, row[col.field])" :class="formatClass(col, row[col.field])">
                <slot :name="'col-'+col.field" :row="row" :col="col" :data="row[col.field]">
                  <span v-html="format(row, col)"></span>
                </slot>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </template>

    <div v-else class="q-data-table-container" @mousewheel="mouseWheel" @DOMMouseScroll="mouseWheel">
      <div v-if="hasHeader" class="q-data-table-head" ref="head" :style="{marginRight: scroll.vert}">
        <table-content head :cols="cols" :sorting="sorting" :scroll="scroll" :selection="config.selection" @sort="setSortField"></table-content>
      </div>
      <div
        class="q-data-table-body"
        :style="bodyStyle"
        ref="body"
        @scroll="scrollHandler"
      >
        <div v-if="message" class="q-data-table-message row items-center justify-center" v-html="message"></div>
        <table-content v-else :cols="cols" :selection="config.selection">
          <tr v-for="row in rows" :style="rowStyle">
            <td v-if="config.selection"></td>
            <td v-if="leftStickyColumns" :colspan="leftStickyColumns"></td>
            <td v-for="col in regularCols" :style="formatStyle(col, row[col.field])" :class="formatClass(col, row[col.field])">
              <slot :name="'col-'+col.field" :row="row" :col="col" :data="row[col.field]">
                <span v-html="format(row, col)"></span>
              </slot>
            </td>
            <td v-if="rightStickyColumns" :colspan="rightStickyColumns"></td>
          </tr>
        </table-content>
      </div>

      <template v-if="!message && (leftStickyColumns || config.selection)">
        <div
          class="q-data-table-sticky-left"
          ref="stickyLeft"
          :style="{bottom: scroll.horiz}"
        >
          <table-sticky :no-header="!hasHeader" :sticky-cols="leftStickyColumns" :cols="cols" :sorting="sorting" :selection="config.selection">
            <tr v-for="(row, index) in rows" :style="rowStyle">
              <td v-if="config.selection">
                <q-checkbox v-if="config.selection === 'multiple'" v-model="rowSelection[index]"></q-checkbox>
                <q-radio v-else v-model="rowSelection[0]" :val="index"></q-radio>
              </td>
              <td v-for="col in leftCols" :style="formatStyle(col, row[col.field])" :class="formatClass(col, row[col.field])">
                <slot :name="'col-'+col.field" :row="row" :col="col" :data="row[col.field]">
                  <span v-html="format(row, col)"></span>
                </slot>
              </td>
            </tr>
          </table-sticky>
        </div>
        <div v-if="hasHeader" class="q-data-table-sticky-left" :style="{bottom: scroll.horiz}">
          <table-sticky head :sticky-cols="leftStickyColumns" :scroll="scroll" :cols="cols" :sorting="sorting" @sort="setSortField" :selection="config.selection"></table-sticky>
        </div>
      </template>

      <template v-if="!message && rightStickyColumns">
        <div
          class="q-data-table-sticky-right"
          ref="stickyRight"
          :style="{right: scroll.vert, bottom: scroll.horiz}"
        >
          <table-sticky :no-header="!hasHeader" right :sticky-cols="rightStickyColumns" :cols="cols" :sorting="sorting" :selection="config.selection">
            <tr v-for="row in rows" :style="rowStyle">
              <td v-if="config.selection" class="invisible"></td>
              <td :colspan="cols.length - rightStickyColumns" class="invisible"></td>
              <td v-for="col in rightCols" :style="formatStyle(col, row[col.field])" :class="formatClass(col, row[col.field])">
                <slot :name="'col-'+col.field" :row="row" :col="col" :data="row[col.field]">
                  <span v-html="format(row, col)"></span>
                </slot>
              </td>
            </tr>
          </table-sticky>
        </div>
        <div v-if="hasHeader" class="q-data-table-sticky-right" :style="{right: scroll.vert}">
          <table-sticky right head :sticky-cols="rightStickyColumns" :scroll="scroll" :cols="cols" :sorting="sorting" @sort="setSortField" :selection="config.selection"></table-sticky>
        </div>
      </template>
    </div>

    <table-pagination v-if="config.pagination" :pagination="pagination" :entries="pagination.entries" :labels="labels"></table-pagination>
  </div>
</template>

<script>
import Utils from '../../utils'

import ColumnSelection from './plugins/column-selection/column-selection'
import Filter from './plugins/filter/filter'
import I18n from './plugins/i18n/i18n'
import Pagination from './plugins/pagination/pagination'
import Responsive from './plugins/responsive/responsive'
import RowSelection from './plugins/row-selection/row-selection'
import Scroll from './plugins/scroll/scroll'
import Sort from './plugins/sort/sort'
import StickyColumns from './plugins/sticky-cols/sticky-cols'

import TableContent from './TableContent.vue'

export default {
  mixins: [ColumnSelection, Filter, I18n, Pagination, Responsive, RowSelection, Scroll, Sort, StickyColumns],
  props: {
    data: {
      type: Array,
      default: []
    },
    columns: {
      type: Array,
      required: true
    },
    config: {
      type: Object,
      default () { return {} }
    }
  },
  data () {
    return {
      selected: false,
      toolbar: '',
      refreshing: false
    }
  },
  computed: {
    rows () {
      let length = this.data.length

      if (!length) {
        return []
      }

      let rows = Utils.clone(this.data)

      rows.forEach((row, i) => {
        row.__index = i
      })

      if (this.filtering.terms) {
        rows = this.filter(rows)
      }

      if (this.sorting.field) {
        this.sort(rows)
      }

      this.pagination.entries = rows.length
      if (this.pagination.rowsPerPage > 0) {
        rows = this.paginate(rows)
      }

      return rows
    },
    rowStyle () {
      if (this.config.rowHeight) {
        return {height: this.config.rowHeight}
      }
    },
    bodyStyle () {
      return this.config.bodyStyle || {}
    },
    hasToolbar () {
      return this.config.title || this.filteringCols.length || this.config.columnPicker || this.config.refresh
    },
    hasHeader () {
      return !this.config.noHeader
    }
  },
  methods: {
    resetBody () {
      let body = this.$refs.body

      if (body) {
        body.scrollTop = 0
      }
      this.pagination.page = 1
    },
    format (row, col) {
      return col.format ? col.format(row[col.field], row) : row[col.field]
    },
    refresh (state) {
      if (state === false) {
        this.refreshing = false
      }
      else if (state === true || !this.refreshing) {
        this.refreshing = true
        this.$emit('refresh', () => {
          this.refreshing = false
        })
      }
    },
    formatStyle (col, value) {
      return typeof col.style === 'function' ? col.style(value) : col.style
    },
    formatClass (col, value) {
      return typeof col.classes === 'function' ? col.classes(value) : col.classes
    }
  },
  components: {
    TableContent
  }
}
</script>
