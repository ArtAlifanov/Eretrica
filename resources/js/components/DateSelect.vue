<template>
  <div class="custom-select">
    <el-dropdown
      ref="dropdown"
      trigger="click"
      placement="bottom"
      @visible-change="handleVisibleChange"
    >
      <el-button @click="handleDropdownClick" :class="[{ 'selected-button': getKeyFrom() }]">
        <span v-if="!getKeyFrom()">{{ filter.name }}</span>
        <el-tag :key="selectedItem.id" size="mini" :closable="true" @close="handleSelect()" v-else>
          <span v-if="filter.name">{{ `${filter.name}: ` }}</span>
          {{ ` ${getKeyFrom()}` }}
          <span v-if="getKeyTo()">{{ `-${getKeyTo()} ` }}</span>
        </el-tag>
      </el-button>
      <el-button
        v-if="keyFrom"
        icon="el-icon-close"
        class="close-button"
        @click.prevent.stop="handleReset(true)"
      ></el-button>
      <el-dropdown-menu slot="dropdown">
        <template>
          <div class="dropdown-container">
            <!-- <el-select
              v-model="selectedItem"
              :placeholder="$t('general.date_range.custom')"
              @change="datePickerChange(filter)"
            >
              <el-option
                v-for="(shortcut, key) in shortcuts"
                :key="key"
                :label="shortcut.text"
                :value="key"
              ></el-option>
            </el-select>-->

            <div class="custom-date-picker">
              <!-- <el-button
                  ref="pickerButton"
                  @click="handleClick"
                  :class="[{'selected-button': selectedDate !== ''}]"
                >
                  <span>{{ filter.name }}</span>
                  <el-tag v-if="selectedDate !== ''" size="mini">{{ `: ${selectedDate}` }}</el-tag>
                </el-button>
                <el-button
                  v-if="selectedDate !== ''"
                  icon="el-icon-close"
                  class="close-button"
                  @click.prevent.stop="handleReset(true)"
              ></el-button>-->

              <custom-date-picker
                ref="datePicker"
                :format="filter.format"
                :placeholder="filter.name"
                :value-format="filter.format"
                style="width: 100%"
                type="daterange"
                :refOfDropdown="$refs"
                :isDropdownVisible= "isDropdownVisible"
                align="center"
                v-model="selectedDate"
                popper-class="select-date-panel"
                :picker-options="pickerOptions"
              ></custom-date-picker>
            </div>
          </div>

          <el-divider></el-divider>
          <div class="actions">
            <el-button type="text" @click="handleReset()">
              <el-dropdown-item>
                {{
                $t("general.reset")
                }}
              </el-dropdown-item>
            </el-button>
            <el-button v-if="!getSelectedDate()" type="text" :disabled="true">
              <el-dropdown-item>{{ $t("general.placeholders.select") }}</el-dropdown-item>
            </el-button>
            <el-button v-else type="primary" @click="handleSelect()">
              <el-dropdown-item>{{$t("general.placeholders.select")}}</el-dropdown-item>
            </el-button>
          </div>
        </template>
      </el-dropdown-menu>
    </el-dropdown>
    <span :id="filter.type + filter.name" style="visibility: hidden; position: absolute"></span>
  </div>
</template>
<script>
import {
  subDays,
  format,
  startOfWeek,
  lastDayOfWeek,
  subMonths,
  startOfMonth,
  endOfMonth,
  lastDayOfMonth
} from "date-fns";

import CustomDatePicker from "./customDatepicker";

export default {
  components: { CustomDatePicker },

  data() {
    return {
      isDropdownVisible:false,
      originItems: [],
      shortcuts: [],
      selecting: false,
      selectedItem: "",
      items: [],
      selectedDate: [],
      forceRecompute: false,
      keyFrom: "",
      keyTo: ""
    };
  },
  props: {
    filter: {
      type: Object,
      default: () => {}
    },
    selectedOptions: {
      type: Array,
      default: () => []
    }
  },
  computed: {
    pickerOptions: function() {
      let custom = {
        text: this.$t('general.date_range.custom'),
      };
      let select = {
        text: this.$t('general.placeholders.select'),
      };
      let lastWeek = {
        text: this.$t("general.date_range.last_week"),
        start: "",
        end: "",
        onClick(picker) {
          const end = subDays(new Date(), 7);
          this.start = startOfWeek(end);
          this.end = lastDayOfWeek(end);

          picker.$emit("pick", [this.start, this.end]);
        }
      };
      let last7Days = {
        text: this.$t("general.date_range.last_7_days"),
        start: "",
        end: "",
        onClick(picker) {
          this.end = new Date();
          this.start = new Date();
          this.start.setTime(this.start.getTime() - 3600 * 1000 * 24 * 7);
          picker.$emit("pick", [this.start, this.end]);
        }
      };
      let last14Days = {
        text: this.$t("general.date_range.last_14_days"),
        start: "",
        end: "",
        onChange(picker) {
          this.start = new Date();
          this.end = new Date();
          this.start.setTime(this.start.getTime() - 3600 * 1000 * 24 * 14);
          picker.$emit("pick", [this.start, this.end]);
        }
      };
      let last30Days = {
        text: this.$t("general.date_range.last_30_days"),
        start: "",
        end: "",
        onClick(picker) {
          this.end = new Date();
          this.start = new Date();
          this.start.setTime(this.start.getTime() - 3600 * 1000 * 24 * 30);
          picker.$emit("pick", [this.start, this.end]);
        }
      };
      let lastMonth = {
        text: this.$t("general.date_range.last_month"),
        start: "",
        end: "",
        onClick(picker) {
          let end = subMonths(new Date(), 1);
          this.start = startOfMonth(end);
          this.end = lastDayOfMonth(end);

          picker.$emit("pick", [this.start, this.end]);
        }
      };
      let last3Months = {
        text: this.$t("general.date_range.last_3_months"),
        start: "",
        end: "",
        onClick(picker) {
          this.end = new Date();
          this.start = new Date();
          this.start.setTime(this.start.getTime() - 3600 * 1000 * 24 * 90);
          picker.$emit("pick", [this.start, this.end]);
        }
      };
      let last6Months = {
        text: this.$t("general.date_range.last_6_months"),
        start: "",
        end: "",
        onClick(picker) {
          this.end = new Date();
          this.start = new Date();
          this.start.setTime(this.start.getTime() - 3600 * 1000 * 24 * 183);
          picker.$emit("pick", [this.start, this.end]);
        }
      };
      let lastYear = {
        text: this.$t("general.date_range.last_year"),
        start: "",
        end: "",
        onClick(picker) {
          this.end = new Date();
          this.start = new Date();
          this.start.setTime(this.start.getTime() - 3600 * 1000 * 24 * 365);
          picker.$emit("pick", [this.start, this.end]);
        }
      };
      let last2Years = {
        text: this.$t("general.date_range.last_2_years"),
        start: "",
        end: "",
        onClick(picker) {
          this.end = new Date();
          this.start = new Date();
          this.start.setTime(this.start.getTime() - 3600 * 1000 * 24 * 730);
          picker.$emit("pick", [this.start, this.end]);
        }
      };

      return {
        shortcuts: [select, custom, last7Days, last30Days, lastWeek, lastMonth, last3Months]
        /*  disabledDate(time) {
          return time.getTime() > Date.now();
        }*/
      };
    }
  },
  methods: {
    // datePickerChange(filter) {
    //   this.keyFrom = this.selectedDate[0];
    //   this.keyTo = this.selectedDate[1];

    //   this.$refs.dropdown.$emit("visible-change", true);
    // },
    isContained(str) {
      if (str === undefined) return false;
      return str.toLowerCase().includes(this.search.toLowerCase());
    },
    handleDropdownClick() {
      this.handleSelect();
    },
    handleVisibleChange(visible) {
      this.isDropdownVisible = visible
      let selected = [],
        unselected = [];
      if (visible) {
        // this.$refs.datePicker.focus();
        this.selecting = visible;
        if (!this.showGroup) {
          this.items.forEach(item => {
            if (item.selected) {
              selected.push(item);
            } else {
              unselected.push(item);
            }
          });
          unselected.sort((a, b) => {
            return a.id - b.id;
          });
          this.items = selected.concat(unselected);
        }
        this.originItems = [];
        this.items.forEach(item => {
          this.originItems.push({
            id: item.id,
            name: item.name,
            selected: item.selected
          });
        });
      } else this.handleSelect();
    },
    getSelectedDate() {
      let result = "";
      if (this.selectedDate.length != 0) result = this.selectedDate;
      return result;
    },

    getKeyFrom() {
      let result = "";
      if (this.keyFrom) result = this.keyFrom;
      return result;
    },
    getKeyTo() {
      let result = "";
      if (this.keyTo) result = this.keyTo;
      return result;
    },
    handleReset(notifyChange = false) {
      this.selectedDate = "";
      this.handleSelect();
    },
    handleClick() {},
    handleSelect() {
      let result = {};
      if (this.selectedDate) {
        this.keyFrom = this.selectedDate[0];
        this.keyTo = this.selectedDate[1];
      } else {
        this.keyFrom = "";
        this.keyTo = "";
      }
      result.from = this.keyFrom;
      if (this.keyFrom && !this.keyTo) {
        result.to = this.keyFrom;
      } else {
        result.to = this.keyTo;
      }

      this.$emit("date-changed", result);
    }
  },
  created() {},
  mounted() {
    const {created_from, created_to} = this.$route.query;
      if(created_from && created_to){
        this.keyFrom = created_from;
        this.keyTo = created_to;
        let selectedDate = [created_from, created_to];
        this.selectedDate = selectedDate;
      }
    this.forceRecompute = true;
  },
  // watch: {
  //   selectedDate() {
  //     if (this.selectedDate) {
  //       this.keyFrom = this.selectedDate[0];
  //       this.keyTo = this.selectedDate[1];
  //     } else {
  //       this.keyFrom = "";
  //       this.keyTo = "";
  //     }
  //     //this.handleSelect();
  //   }
  // },
  updated() {}
};
</script>
<style lang="scss" scoped>
.el-dropdown {
  width: 100%;
  position: relative;
  height: 40px;
  .el-button {
    padding: 0 15px;
    width: 100%;
    text-align: left;
    color: #3d3f41;
    height: 35px;
    position: relative;
    border-color: transparent;
    border-radius: 6px;
    background-color: #f0f0f0;
    &:hover,
    &:focus {
     // background-color: var(--color-main-background-base);
      box-shadow: none;
      background-color: var(--color-primary);
      color: #fff
    }
    &.selected {
      border-color: 1px solid var(--color-text-primary);
      background-color: var(--color-text-primary);
    }

    span.el-tag {
      padding: 0 5px !important;
      &:not(:last-of-type) {
        margin-right: 5px;
      }
      border-color: transparent;
      background-color: var(--color-primary);
      color: var(--color-white);
      line-height: 30px;
      height: 30px;

      :global(i.el-tag__close) {
        display: none;
      }
      &.select-count {
        padding: 0 4px !important;
        margin-right: 5px;
        height: 20px;
        line-height: 20px;
        text-align: center;
        border-radius: 4px;
        border: 1px solid var(--color-white);
      }
      &.select-count.multi-tag {
        position: absolute;
        right: 5px;
        top: 5px;
        padding: 0 7px !important;
        margin-right: 0px;
        height: 30px;
        line-height: 30px;
        text-align: center;
        border-radius: 4px;
        border: 1px solid var(--color-white);
      }
    }
    &.multi-tag {
      padding: 0 5px;
    }
    &.selected-button {
      background-color: var(--color-primary);
      padding: 0 45px 0 5px;
      .el-tag {
        padding: 0px;
      }
    }
  }
  .close-button {
    position: absolute;
    right: 0;
    top: 0;
    width: auto;
    margin: 0;
    padding: 0 10px;
    font-size: 20px;
    border-radius: 0 4px 4px 0;
    border-left: 1px solid var(--color-white);
    background-color: transparent;
    color: white;
    &:hover {
      background-color: transparent;
    }
  }
}
.el-dropdown-menu {
  margin-bottom: 0px;
  padding: 0px;
  width: 288px;
  border-radius: 12px;
  &.is-hide {
    visibility: hidden;
  }
  &.group-show {
    width: 320px;
  }
  .el-input {
    margin: 16px 16px 0;
    width: calc(100% - 32px);
    :global(.el-input__inner) {
      border-color: transparent;
      color: var(--color-text-regular);
      background-color: var(--background-color-base);
      height: 36px;
    }
    :global(.el-input__icon) {
      font-size: 16px;
      line-height: 36px;
    }
  }
  .dropdown-container {
    max-height: 445px;
    overflow-y: auto;

    &::-webkit-scrollbar {
      width: 8px;
    }
    &::-webkit-scrollbar-thumb {
      background-color: var(--color-text-placeholder);
      border: 1px solid transparent;
      border-radius: 11px;
      background-clip: content-box;
    }
    &::-webkit-scrollbar * {
      background: transparent;
    }

    .dropdown-item {
      padding: 4px;
      &:not(:last-child) {
        margin-bottom: 6px;
      }
      border: 1px solid var(--border-color-base);
      border-radius: 4px;
      cursor: pointer;
      position: relative;
      display: flex;
      justify-content: space-between;

      span {
        user-select: none;
      }
      span.el-icon-check {
        display: none;
        position: absolute;
        right: 8px;
        top: 8px;
      }
      &:hover {
        background-color: var(--background-color-base);
      }
      &.selected {
        border: 1px solid var(--color-black);
        display: flex;
        justify-content: space-between;
        align-items: center;
        span {
          font-weight: 700;
          &:first-child {
            margin-right: 15px;
          }
        }
        span.item-type {
          padding-right: 25px;
        }
        span.el-icon-check {
          display: block;
        }
      }
      &.hide-unmatching {
        display: none;
      }
    }
    .dropdown-item-label {
      padding: 0 4px;
      margin-bottom: 5px;
      span {
        color: var(--color-text-placeholder);
      }
      &:not(:nth-of-type(1)) {
        margin-top: 15px;
      }

      &.hide-unmatching {
        display: none;
      }
    }
  }
  .el-divider {
    margin: 16px 0px 0px;
  }
  .actions {
    padding: 8px 16px;
    text-align: right;
    .el-button--text {
      &:hover {
        color: var(--primary-color);
      }
    }
    .el-button {
      width: 50%;
      padding: 0;

      .el-dropdown-menu__item {
        list-style: none;
        line-height: 36px;
        padding: 0 20px;
        margin: 0;
        font-size: 14px;
        color: var(--color-text-regular);
        cursor: pointer;
        outline: 0;
      }

      &:first-child {
        width: 40%;
        .el-dropdown-menu__item:hover {
          background-color: transparent;
        }
      }
      &:last-child {
        .el-dropdown-menu__item {
          color: inherit;
        }
      }
    }
  }

  &.el-popper {
    :global(&[x-placement^="bottom"] .popper__arrow) {
      top: -12px;
      &::after {
        margin-left: -10px;
      }
    }
    :global(.popper__arrow) {
      border-bottom-width: 12px;
      border-left-width: 10px;
      border-right-width: 10px;
      border-top-width: 0;
      &::after {
        border-width: 12px;
        border-left-width: 10px;
        border-right-width: 10px;
        border-top-width: 0;
      }
    }
  }
}
</style>
<style lang="scss" scoped>
.el-dropdown-menu .dropdown-container {
  max-height: auto;
  overflow: hidden;
}
.select-date-panel {
  &.el-date-range-picker {
    box-shadow: none;
    border: none;
    width: 100%;
  }
}
</style>
