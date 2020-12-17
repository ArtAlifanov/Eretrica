<template>
  <transition name="el-zoom-in-top" @after-leave="$emit('dodestroy')">
    <div
      v-show="visible"
      class="el-picker-panel el-date-range-picker el-popper"
      :class="[{
        'has-sidebar': $slots.sidebar || shortcuts
      }, popperClass]"
    >
      <div class="el-picker-panel__body-wrapper">
        <el-select
          v-if="shortcuts"
          v-model="selectedItem"
          @change="shortcutChanged()"
        >
          <el-option
            v-for="(shortcut, key) in shortcuts"
            :key="key"
            :label="shortcut.text"
            :value="key"
          ></el-option>
        </el-select>
        <slot name="sidebar" class="el-picker-panel__sidebar"></slot>
        <div class="el-picker-panel__sidebar">
          <button
            type="button"
            class="el-picker-panel__shortcut"
            v-for="(shortcut, key) in shortcuts"
            :key="key"
            ref="shortcut"
            @click="handleShortcutClick(shortcut)"
          >{{shortcut.text}}</button>
        </div>

        <el-row v-if="isCustomSelect">
          <el-col :span="12">
            <div class="grid-content">
              <el-input
                :placeholder="$t('el.datepicker.startDate')"
                :value="minVisibleDate"
                :class="[{'is-active':getSelectedInput('min'),'min-value-selected':minVisibleDate}]"
                @click="handleInput('min')"
                @focus="handleInput('min')"
                @input="val => handleDateInput(val, 'min')"
                @change="val => handleDateChange(val, 'min')"
              ></el-input>
              <span class="separator">
                <span>{{$t('general.date_range.range_separator')}}</span>
              </span>
            </div>
          </el-col>
          <el-col :span="12">
            <div class="grid-content">
              <el-input
                :placeholder="$t('el.datepicker.endDate')"
                :value="maxVisibleDate"
                :readonly="!minDate"
                :class="[{'is-active':getSelectedInput('max'),'max-value-selected':maxVisibleDate}]"
                @click="handleInput('max')"
                @focus="handleInput('max')"
                @input="val => handleDateInput(val, 'max')"
                @change="val => handleDateChange(val, 'max')"
              ></el-input>
            </div>
          </el-col>
        </el-row>

        <div v-if="isCustomSelect" class="el-picker-panel__body">
          <div class="el-picker-panel__content el-date-range-picker__content is-left">
            <div class="el-date-range-picker__header">
              <button
                type="button"
                @click="leftPrevYear"
                class="el-picker-panel__icon-btn el-icon-d-arrow-left"
              ></button>
              <button
                type="button"
                @click="leftPrevMonth"
                class="el-picker-panel__icon-btn el-icon-arrow-left"
              ></button>
              <button
                type="button"
                @click="leftNextYear"
                class="el-picker-panel__icon-btn el-icon-d-arrow-right"
              ></button>
              <button
                type="button"
                @click="leftNextMonth"
                class="el-picker-panel__icon-btn el-icon-arrow-right"
              ></button>
              <div>{{ leftLabel }}</div>
            </div>
            <date-table
              selection-mode="range"
              :date="leftDate"
              :default-value="defaultValue"
              :min-date="minDate"
              :max-date="maxDate"
              :range-state="rangeState"
              :disabled-date="disabledDate"
              :cell-class-name="cellClassName"
              @changerange="handleChangeRange"
              :first-day-of-week="firstDayOfWeek"
              @pick="handleRangePick"
            ></date-table>
          </div>
        </div>
      </div>
    </div>
  </transition>
</template>

<script type="text/babel">
import {
  formatDate,
  parseDate,
  isDate,
  modifyDate,
  modifyTime,
  modifyWithTimeString,
  prevYear,
  nextYear,
  prevMonth,
  nextMonth,
  nextDate,
  extractDateFormat,
  extractTimeFormat
} from "element-ui/src/utils/date-util";
import Clickoutside from "element-ui/src/utils/clickoutside";
import DateTable from "./dateTable";
import ElInput from "element-ui/packages/input";
import ElButton from "element-ui/packages/button";

const calcDefaultValue = defaultValue => {
  if (Array.isArray(defaultValue)) {
    return [new Date(defaultValue[0]), new Date(defaultValue[1])];
  } else if (defaultValue) {
    return [new Date(defaultValue), nextDate(new Date(defaultValue), 1)];
  } else {
    return [new Date(), nextDate(new Date(), 1)];
  }
};

export default {
  //  mixins: [Locale],

  directives: { Clickoutside },

  computed: {
    leftLabel() {
      return (
        this.leftDate.getFullYear() +
        " " +
        this.$t("el.datepicker.year") +
        " " +
        this.$t(`el.datepicker.month${this.leftDate.getMonth() + 1}`)
      );
    },

    rightLabel() {
      return (
        this.rightDate.getFullYear() +
        " " +
        this.$t("el.datepicker.year") +
        " " +
        this.$t(`el.datepicker.month${this.rightDate.getMonth() + 1}`)
      );
    },

    leftYear() {
      return this.leftDate.getFullYear();
    },

    leftMonth() {
      return this.leftDate.getMonth();
    },

    leftMonthDate() {
      return this.leftDate.getDate();
    },

    rightYear() {
      return this.rightDate.getFullYear();
    },

    rightMonth() {
      return this.rightDate.getMonth();
    },

    rightMonthDate() {
      return this.rightDate.getDate();
    },

    minVisibleDate() {
      if (this.dateUserInput.min !== null) return this.dateUserInput.min;
      if (this.minDate) return formatDate(this.minDate, this.dateFormat);
      return "";
    },
    maxVisibleDate() {
      if (this.dateUserInput.max !== null) return this.dateUserInput.max;
      if (this.maxDate)
        return formatDate(this.maxDate, this.dateFormat);
      return "";
    },

    dateFormat() {
      if (this.format) {
        return extractDateFormat(this.format);
      } else {
        return "yyyy-MM-dd";
      }
    },

    enableMonthArrow() {
      const nextMonth = (this.leftMonth + 1) % 12;
      const yearOffset = this.leftMonth + 1 >= 12 ? 1 : 0;
      return (
        new Date(this.leftYear + yearOffset, nextMonth) <
        new Date(this.rightYear, this.rightMonth)
      );
    },

    enableYearArrow() {
      return (
        this.rightYear * 12 +
          this.rightMonth -
          (this.leftYear * 12 + this.leftMonth + 1) >=
        12
      );
    }
  },
  props:{
    refOfDropdown:Object,
    isDropdownVisible:Boolean,
  },
  data() {
    return {
      isCustomSelect:false,
      popperClass: "",
      value: [],
      defaultValue: null,
      defaultTime: null,
      minDate: "",
      maxDate: "",
      leftDate: new Date(),
      rightDate: nextMonth(new Date()),
      rangeState: {
        endDate: null,
        selecting: false,
        row: null,
        column: null
      },
      selectedItem: 0,
      selectedMin: false,
      selectedMax: false,
      showTime: false,
      shortcuts: "",
      visible: "",
      disabledDate: "",
      cellClassName: "",
      firstDayOfWeek: 7,
      format: "",

      arrowControl: false,
      dateUserInput: {
        min: null,
        max: null
      }
    };
  },

  watch: {
    isDropdownVisible(value){
      if(!value){
        this.isCustomSelect = false
      }
    },
    minDate(val) {
      this.dateUserInput.min = null;
      /* this.$nextTick(() => {
        if (this.maxDate) {
          this.value = [this.minDate, this.maxDate];
        }
      });*/
    },

    maxDate(val) {
      this.dateUserInput.max = null;
      //this.value = [this.minDate, this.maxDate];
    },
    value(newVal) {
      if (!newVal) {
        this.minDate = null;
        this.maxDate = null;
      } else if (Array.isArray(newVal)) {
        // this.minDate = isDate(newVal[0]) ? new Date(newVal[0]) : null;
        // this.maxDate = isDate(newVal[1]) ? new Date(newVal[1]) : null;
        if (this.minDate) {
          this.leftDate = this.minDate;
          if (this.maxDate) {
            const minDateYear = this.minDate.getFullYear();
            const minDateMonth = this.minDate.getMonth();
            const maxDateYear = this.maxDate.getFullYear();
            const maxDateMonth = this.maxDate.getMonth();
            this.rightDate =
              minDateYear === maxDateYear && minDateMonth === maxDateMonth
                ? nextMonth(this.maxDate)
                : this.maxDate;
          } else {
            this.rightDate = nextMonth(this.leftDate);
          }
        } else {
          this.leftDate = calcDefaultValue(this.defaultValue)[0];
          this.rightDate = nextMonth(this.leftDate);
        }
      }
    },

    defaultValue(val) {
      if (!Array.isArray(this.value)) {
        const [left, right] = calcDefaultValue(val);
        this.leftDate = left;
        this.rightDate = val && val[1] ? right : nextMonth(this.leftDate);
      }
    }
  },
  mounted() {
    const {created_from, created_to} = this.$route.query;
      if(created_from && created_to){
        let min = created_from.split(".");
        let max = created_to.split(".");  
        let minDate = `${min[1]}-${min[0]}-${min[2]}`
        let maxDate = `${max[1]}-${max[0]}-${max[2]}`
        this.minDate = new Date(minDate)
        this.maxDate = new Date(maxDate)
      }
  },

  methods: {
    getSelectedInput(type) {
      if (type === "min") {
        return this.selectedMin;
      }
      return this.selectedMax;
    },

    handleClear() {
      this.minDate = null;
      this.maxDate = null;
      this.leftDate = calcDefaultValue(this.defaultValue)[0];
      this.rightDate = nextMonth(this.leftDate);
      this.$emit("pick", null);
    },
    shortcutChanged() {
      this.$refs.shortcut[this.selectedItem].click();
      this.selectedItem = 0;
    },
    handleChangeRange(val) {
      this.minDate = val.minDate;
      this.maxDate = val.maxDate;
      this.rangeState = val.rangeState;
    },
    handleInput(type) {
      if (type === "min") {
        this.selectedMax = false;
        this.selectedMin = true;
      } else {
        this.selectedMin = false;
        this.selectedMax = true;
      }
    },

    handleDateInput(value, type) {
      this.dateUserInput[type] = value;
      if (value.length !== this.dateFormat.length) return;
      const parsedValue = parseDate(value, this.dateFormat);

      if (parsedValue) {
        if (
          typeof this.disabledDate === "function" &&
          this.disabledDate(new Date(parsedValue))
        ) {
          return;
        }
        if (type === "min") {
          this.minDate = modifyDate(
            this.minDate || new Date(),
            parsedValue.getFullYear(),
            parsedValue.getMonth(),
            parsedValue.getDate()
          );
          this.leftDate = new Date(parsedValue);
        } else {
          this.maxDate = modifyDate(
            this.maxDate || new Date(),
            parsedValue.getFullYear(),
            parsedValue.getMonth(),
            parsedValue.getDate()
          );
          this.rightDate = new Date(parsedValue);
        }
      }
    },

    handleDateChange(value, type) {
      const parsedValue = parseDate(value, this.dateFormat);
      if (parsedValue) {
        if (type === "min") {
          this.minDate = modifyDate(
            this.minDate,
            parsedValue.getFullYear(),
            parsedValue.getMonth(),
            parsedValue.getDate()
          );
          if (this.minDate > this.maxDate) {
            this.maxDate = this.minDate;
          }
        } else {
          this.maxDate = modifyDate(
            this.maxDate,
            parsedValue.getFullYear(),
            parsedValue.getMonth(),
            parsedValue.getDate()
          );
          if (this.maxDate < this.minDate) {
            this.minDate = this.maxDate;
          }
          this.handleConfirm();
        }
      }
    },

    handleRangePick(val, close = true) {
      const defaultTime = this.defaultTime || [];
      const minDate = modifyWithTimeString(val.minDate, defaultTime[0]);
      const maxDate = modifyWithTimeString(val.maxDate, defaultTime[1]);

      if (this.maxDate === maxDate && this.minDate === minDate) {
        return;
      }

      this.onPick && this.onPick(val);
      let selectedDate = this.rangeState.selecting ? maxDate : minDate;

      if (this.selectedMin) {
        this.minDate = selectedDate;
        this.selectedMin = false;
        if (this.minDate > this.maxDate) {
          this.maxDate = this.minDate;
        }
        setTimeout(() => {
          this.minDate = selectedDate;
          this.selectedMin = false;
          if (this.minDate > this.maxDate) {
            this.maxDate = this.minDate;
          }
        }, 10);
      } else if (this.selectedMax) {
        this.maxDate = selectedDate;
        this.selectedMax = false;
        if (this.minDate > this.maxDate) {
          this.maxDate = this.minDate;
        }
        setTimeout(() => {
          this.maxDate = selectedDate;
          this.selectedMax = false;
          if (this.minDate > this.maxDate) {
            this.maxDate = this.minDate;
          }
        }, 10);
      } else {
        this.maxDate = maxDate;
        this.minDate = minDate;
        setTimeout(() => {
          this.maxDate = maxDate;
          this.minDate = minDate;
        }, 10);
      }

      // workaround for https://github.com/ElemeFE/element/issues/7539, should remove this block when we don't have to care about Chromium 55 - 57

      this.handleConfirm();
    },

    handleShortcutClick(shortcut) {
      let result = {
        minDate: shortcut.start,
        maxDate: null,
        rangeState: {
          endDate: shortcut.end,
          selecting: false,
          row: null,
          column: null
        }
      };

      if (shortcut.onClick) {
        this.isCustomSelect = false;
        shortcut.onClick(this);
        this.handleChangeRange(result);
        this.handleChangeRange(result);
      }else{
        this.isCustomSelect = true;
        this.refOfDropdown.dropdown.show()
      }
    },

    leftPrevYear() {
      this.leftDate = prevYear(this.leftDate);
    },

    leftPrevMonth() {
      this.leftDate = prevMonth(this.leftDate);
    },

    rightNextYear() {
      this.rightDate = nextYear(this.rightDate);
    },

    rightNextMonth() {
      this.rightDate = nextMonth(this.rightDate);
    },

    leftNextYear() {
      this.leftDate = nextYear(this.leftDate);
    },

    leftNextMonth() {
      this.leftDate = nextMonth(this.leftDate);
    },

    rightPrevYear() {
      this.rightDate = prevYear(this.rightDate);
    },

    rightPrevMonth() {
      this.rightDate = prevMonth(this.rightDate);
    },

    handleConfirm(visible = false) {
      if (this.isValidValue([this.minDate, this.maxDate])) {
        this.$emit("pick", [this.minDate, this.maxDate], visible);
      }
    },

    isValidValue(value) {
      return (
        Array.isArray(value) &&
        value &&
        value[0] &&
        value[1] &&
        isDate(value[0]) &&
        isDate(value[1]) &&
        value[0].getTime() <= value[1].getTime() &&
        (typeof this.disabledDate === "function"
          ? !this.disabledDate(value[0]) && !this.disabledDate(value[1])
          : true)
      );
    },

    resetView() {
      // NOTE: this is a hack to reset {min, max}Date on picker open.
      // TODO: correct way of doing so is to refactor {min, max}Date to be dependent on value and internal selection state
      //       an alternative would be resetView whenever picker becomes visible, should also investigate date-panel's resetView
      if (this.minDate && this.maxDate == null)
        this.rangeState.selecting = false;
      this.minDate =
        this.value && isDate(this.value[0]) ? new Date(this.value[0]) : null;
      this.maxDate =
        this.value && isDate(this.value[0]) ? new Date(this.value[1]) : null;
    }
  },

  components: { DateTable, ElInput, ElButton }
};
</script>
<style lang="scss" scoped>
.el-date-range-picker.has-sidebar {
  box-shadow: none;
  border: none;
  width: 100%;
}
.el-picker-panel__sidebar {
  display: none;
}
.el-picker-panel__sidebar + .el-picker-panel__body {
  margin-left: 0;
  width: 100%;
  min-width: 288px;
}
.el-date-range-picker__content {
  &.is-left {
    width: 100%;
    border: none;
  }
  &.is-right {
    display: none;
    border: none;
  }
}
.el-date-range-picker .el-picker-panel__body {
  min-width: 288px;
}
.el-select,
.el-input {
  margin: 16px 16px 0;
  width: calc(100% - 32px);
}
.grid-content {
  position: relative;
}
.separator {
  position: absolute;
  right: -16px;
  top: 21px;
  text-transform: lowercase;
  text-align: center;
  width: 32px;
}
.el-date-table {
  :global(td.end-date span) {
    background-color: transparent;
  }

  :global(td.start-date span) {
    background-color: transparent;
  }
  :global(td div) {
    background-color: transparent;
    border-radius: 15px;
    margin-left: 0;
    margin-right: 0;
  }
  :global(td.available div) {
    color: #525252;
  }
  :global(td div:hover) {
    background-color: var(--primary-color);
    color: #fff;
  }
  :global(td.today span:hover) {
    color: #fff;
  }
}
  :global(.min-value-selected  .el-input__inner) {
    background:#f1f0f1 !important;
  }
  :global(.max-value-selected  .el-input__inner) {
      background:#f1f0f1 !important;
  }
 
</style>