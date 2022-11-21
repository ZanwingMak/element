<template>
  <transition name="el-zoom-in-top" @after-leave="$emit('dodestroy')">
    <div
      v-show="visible"
      class="el-picker-panel el-date-range-picker el-popper"
      :class="[{
        'has-sidebar': $slots.sidebar || shortcuts
      }, popperClass]">
      <div class="el-picker-panel__body-wrapper">
        <slot name="sidebar" class="el-picker-panel__sidebar"></slot>
        <div class="el-picker-panel__sidebar" v-if="shortcuts">
          <button
            type="button"
            class="el-picker-panel__shortcut"
            v-for="(shortcut, key) in shortcuts"
            :key="key"
            @click="handleShortcutClick(shortcut)">{{shortcut.text}}</button>
        </div>
        <div class="el-picker-panel__body">
          <div class="el-picker-panel__content el-date-range-picker__content is-left">
            <div class="el-date-range-picker__header">
              <button
                type="button"
                @click="leftPrevYear"
                class="el-picker-panel__icon-btn el-icon-d-arrow-left"></button>
              <button
                type="button"
                v-if="unlinkPanels"
                @click="leftNextYear"
                :disabled="!enableYearArrow"
                :class="{ 'is-disabled': !enableYearArrow }"
                class="el-picker-panel__icon-btn el-icon-d-arrow-right"></button>
              <div>{{ leftLabel }}</div>
            </div>
            <month-table
              ref="leftDate"
              key="leftDate"
              table-key="leftDate"
              table-type="min"
              selection-mode="range"
              :date="leftDate"
              :default-value="defaultValue"
              :min-date="minDate"
              :max-date="maxDate"
              :range-state="rangeState"
              :disabled-date="disabledDate"
              @changeminrange="handleMinChangeRange"
              @minpick="handleMinRangePick"
              @pick="handleRangePick">
            </month-table>
          </div>
          <div class="el-picker-panel__content el-date-range-picker__content is-right">
            <div class="el-date-range-picker__header">
              <button
                type="button"
                v-if="unlinkPanels"
                @click="rightPrevYear"
                :disabled="!enableYearArrow"
                :class="{ 'is-disabled': !enableYearArrow }"
                class="el-picker-panel__icon-btn el-icon-d-arrow-left"></button>
              <button
                type="button"
                @click="rightNextYear"
                class="el-picker-panel__icon-btn el-icon-d-arrow-right"></button>
              <div>{{ rightLabel }}</div>
            </div>
            <month-table
              ref="rightDate"
              key="rightDate"
              table-key="rightDate"
              table-type="max"
              selection-mode="range"
              :date="rightDate"
              :default-value="defaultValue"
              :min-date="minDate"
              :max-date="maxDate"
              :range-state="rangeState"
              :disabled-date="disabledDate"
              @changemaxrange="handleMaxChangeRange"
              @maxpick="handleMaxRangePick"
              @pick="handleRangePick">
            </month-table>
          </div>
        </div>
      </div>
    </div>
  </transition>
</template>

<script type="text/babel">
  import {
    isDate,
    modifyWithTimeString,
    prevYear,
    nextYear,
    nextMonth
  } from 'element-ui/src/utils/date-util';
  import Clickoutside from 'element-ui/src/utils/clickoutside';
  import Locale from 'element-ui/src/mixins/locale';
  import MonthTable from '../basic/month-table';
  import ElInput from 'element-ui/packages/input';
  import ElButton from 'element-ui/packages/button';
  import {isEmpty} from '../../../../src/utils/util';
  import {clearTime} from '../../../../src/utils/date-util';

  const calcDefaultValue = (defaultValue) => {
    if (Array.isArray(defaultValue)) {
      return [new Date(defaultValue[0]), new Date(defaultValue[1])];
    } else if (defaultValue) {
      return [new Date(defaultValue), nextMonth(new Date(defaultValue))];
    } else {
      return [new Date(), nextMonth(new Date())];
    }
  };
  export default {
    mixins: [Locale],

    directives: { Clickoutside },

    computed: {
      btnDisabled() {
        return !(this.minDate && this.maxDate && !this.selecting && this.isValidValue([this.minDate, this.maxDate]));
      },

      leftLabel() {
        return this.leftDate.getFullYear() + ' ' + this.t('el.datepicker.year');
      },

      rightLabel() {
        return this.rightDate.getFullYear() + ' ' + this.t('el.datepicker.year');
      },

      leftYear() {
        return this.leftDate.getFullYear();
      },

      rightYear() {
        return this.rightDate.getFullYear() === this.leftDate.getFullYear() ? this.leftDate.getFullYear() + 1 : this.rightDate.getFullYear();
      },

      enableYearArrow() {
        // note 原来的
        // return this.unlinkPanels && this.rightYear > this.leftYear + 1;
        // note 根据开关配置
        return this.unlinkPanels;
      }
    },

    data() {
      return {
        popperClass: '',
        value: [],
        defaultValue: null,
        defaultTime: null,
        minDate: '',
        maxDate: '',
        leftDate: new Date(),
        // note 原本
        // rightDate: nextYear(new Date()),
        rightDate: new Date(),
        rangeState: {
          endDate: null,
          selecting: false,
          row: null,
          column: null
        },
        shortcuts: '',
        visible: '',
        disabledDate: '',
        format: '',
        arrowControl: false,
        unlinkPanels: false
      };
    },

    watch: {
      value(newVal) {
        if (!newVal) {
          this.minDate = null;
          this.maxDate = null;
        } else if (Array.isArray(newVal)) {
          this.minDate = isDate(newVal[0]) ? new Date(newVal[0]) : null;
          this.maxDate = isDate(newVal[1]) ? new Date(newVal[1]) : null;
          if (this.minDate) {
            this.leftDate = this.minDate;
          }
          if (this.maxDate) {
            this.rightDate = this.maxDate;
          }
          // if (this.minDate) {
          //   this.leftDate = this.minDate;
          //   if (this.unlinkPanels && this.maxDate) {
          //     const minDateYear = this.minDate.getFullYear();
          //     const maxDateYear = this.maxDate.getFullYear();
          //     this.rightDate = minDateYear === maxDateYear
          //       ? nextYear(this.maxDate)
          //       : this.maxDate;
          //   } else {
          //     this.rightDate = nextYear(this.leftDate);
          //   }
          // } else {
          //   this.leftDate = calcDefaultValue(this.defaultValue)[0];
          //   this.rightDate = nextYear(this.leftDate);
          // }
        }
      },

      defaultValue(val) {
        if (!Array.isArray(this.value)) {
          const [left, right] = calcDefaultValue(val);
          this.leftDate = left;
          this.rightDate = right;
          // note 原来的
          // this.rightDate = val && val[1] && left.getFullYear() !== right.getFullYear() && this.unlinkPanels
          //   ? right
          //   : nextYear(this.leftDate);
        }
      }
    },

    methods: {
      handleClear() {
        this.minDate = null;
        this.maxDate = null;
        this.leftDate = calcDefaultValue(this.defaultValue)[0];
        // note 原来的
        // this.rightDate = nextYear(this.leftDate);
        this.rightDate = this.leftDate;
        this.$emit('pick', null);
      },
      handleMinChangeRange(val) {
        this.minDate = val.minDate;
        this.rangeState = val.rangeState;
      },
      handleMaxChangeRange(val) {
        this.maxDate = val.maxDate;
        this.rangeState = val.rangeState;
      },
      handleRangePick(val, close = true) {
        const defaultTime = this.defaultTime || [];
        const minDate = modifyWithTimeString(val.minDate, defaultTime[0]);
        const maxDate = modifyWithTimeString(val.maxDate, defaultTime[1]);
        if (this.maxDate === maxDate && this.minDate === minDate) {
          return;
        }
        this.onPick && this.onPick(val);
        this.maxDate = maxDate;
        this.minDate = minDate;

        // workaround for https://github.com/ElemeFE/element/issues/7539, should remove this block when we don't have to care about Chromium 55 - 57
        setTimeout(() => {
          this.maxDate = maxDate;
          this.minDate = minDate;
        }, 10);
        if (!close) return;
        this.handleConfirm();
      },

      // 选择最小日期抛出
      handleMinRangePick(val, close = true) {
        const defaultTime = this.defaultTime || [];
        const minDate = modifyWithTimeString(val.minDate, defaultTime[0]);
        const maxDate = modifyWithTimeString(val.maxDate, defaultTime[1]);
        if (this.maxDate === maxDate && this.minDate === minDate) {
          return;
        }
        this.onPick && this.onPick(val);
        // this.maxDate = maxDate;
        this.minDate = minDate;

        // workaround for https://github.com/ElemeFE/element/issues/7539, should remove this block when we don't have to care about Chromium 55 - 57
        setTimeout(() => {
          // this.maxDate = maxDate;
          this.minDate = minDate;
        }, 10);
        if (!close || this.showTime) return;
        // NOTE 选中两个日期后不在关闭Popover
        this.handleConfirm(true);
      },
      // 选择最大日期抛出
      handleMaxRangePick(val, close = true) {
        const defaultTime = this.defaultTime || [];
        // const minDate = modifyWithTimeString(val.minDate, defaultTime[0]);
        const maxDate = modifyWithTimeString(val.maxDate, defaultTime[1]);
        // if (this.maxDate === maxDate && this.minDate === minDate) {
        //   return;
        // }
        this.onPick && this.onPick(val);
        this.maxDate = maxDate;
        // this.minDate = minDate;

        // workaround for https://github.com/ElemeFE/element/issues/7539, should remove this block when we don't have to care about Chromium 55 - 57
        setTimeout(() => {
          this.maxDate = maxDate;
          // this.minDate = minDate;
        }, 10);
        if (!close || this.showTime) return;
        // NOTE 选中两个日期后不在关闭Popover
        this.handleConfirm(true);
      },
      handleShortcutClick(shortcut) {
        if (shortcut.onClick) {
          shortcut.onClick(this);
        }
      },

      // leftPrev*, rightNext* need to take care of `unlinkPanels`
      leftPrevYear() {
        this.leftDate = prevYear(this.leftDate);
        if (!this.unlinkPanels) {
          this.rightDate = prevYear(this.rightDate);
        }
      },

      rightNextYear() {
        if (!this.unlinkPanels) {
          this.leftDate = nextYear(this.leftDate);
        }
        this.rightDate = nextYear(this.rightDate);
      },

      // leftNext*, rightPrev* are called when `unlinkPanels` is true
      leftNextYear() {
        this.leftDate = nextYear(this.leftDate);
      },

      rightPrevYear() {
        this.rightDate = prevYear(this.rightDate);
      },

      handleConfirm(visible = false) {
        if (this.isValidValue([this.minDate, this.maxDate])) {
          this.$emit('pick', [this.minDate, this.maxDate], visible);
        }
      },

      isValidValue(value) {
        // note S 修改的
        let status = function(disabledDate, value) {
          // if (!isEmpty(value[0]) && !isEmpty(value[1])) {
          //   return value[0].getTime() <= value[1].getTime();
          // }
          if (!isEmpty(value[0]) || !isEmpty(value[1])) return true;
        };
        return Array.isArray(value) && value && status(this.disabledDate, value);
        // note E 修改的
        // note 原来的
        // return Array.isArray(value) &&
        //   value && value[0] && value[1] &&
        //   isDate(value[0]) && isDate(value[1]) &&
        //   value[0].getTime() <= value[1].getTime() && (
        //   typeof this.disabledDate === 'function'
        //     ? !this.disabledDate(value[0]) && !this.disabledDate(value[1])
        //     : true
        // );
      },
      getDateTimestamp(time) {
        if (typeof time === 'number' || typeof time === 'string') {
          return clearTime(new Date(time)).getTime();
        } else if (time instanceof Date) {
          return clearTime(time).getTime();
        } else {
          return NaN;
        }
      },
      resetView() {
        // NOTE: this is a hack to reset {min, max}Date on picker open.
        // TODO: correct way of doing so is to refactor {min, max}Date to be dependent on value and internal selection state
        //       an alternative would be resetView whenever picker becomes visible, should also investigate date-panel's resetView
        this.minDate = this.value && isDate(this.value[0]) ? new Date(this.value[0]) : null;
        this.maxDate = this.value && isDate(this.value[0]) ? new Date(this.value[1]) : null;
      }
    },

    components: { MonthTable, ElInput, ElButton }
  };
</script>
