<!--
/**
 * @fileoverview Calendar component
 * @license MIT
 * @author Rafal Pospiech <neuronet.io@gmail.com>
 * @package GanttElastic
 */
-->
<template>
  <div
    class="gantt-elastic__calendar-wrapper"
    :style="root.style('calendar-wrapper',{'margin-bottom':root.state.calendar.gap+'px'})"
  >
    <div class="gantt-elastic__calendar" :style="root.style('calendar',{width:getWidth+'px'})">
      <calendar-row :items="root.state.calendar.months" which="month" v-if="root.state.calendar.month.display"></calendar-row>
      <calendar-row :items="root.state.calendar.days" which="day" v-if="root.state.calendar.day.display"></calendar-row>
      <calendar-row :items="root.state.calendar.hours" which="hour" v-if="root.state.calendar.hour.display"></calendar-row>
    </div>
  </div>
</template>

<script>
import dayjs from "dayjs";
import CalendarRow from "./CalendarRow.vue";
export default {
  components: {
    CalendarRow
  },
  inject: ["root"],
  data () {
    return {};
  },

  /**
   * Created
   */
  created () {
    this.root.$on("scope-change", this.regenerate);
    this.root.$on("times-timeZoom-change", this.regenerate);
    this.root.$on("tasks-changed", this.regenerate);
    this.root.$on("options-changed", this.regenerate);
    this.regenerate();
  },

  methods: {
    /**
     * How many hours will fit?
     *
     * @returns {object}
     */
    howManyHoursFit (dayIndex) {
      const stroke = parseFloat(this.root.style('calendar-row-rect')['border-width']);
      const additionalSpace = stroke * 2 + 2;
      let fullCellWidth = this.root.state.times.steps[dayIndex].width.px;
      let formatNames = Object.keys(this.root.state.calendar.hour.format);
      for (let hours = 24; hours > 1; hours = Math.ceil(hours / 2)) {
        for (let formatName of formatNames) {
          if ((this.root.state.calendar.hour.maxWidths[formatName] + additionalSpace) * hours <= fullCellWidth && hours > 1) {
            return {
              count: hours,
              type: formatName
            };
          }
        }
      }
      return {
        count: 0,
        type: ""
      };
    },

    /**
     * How many days will fit?
     *
     * @returns {object}
     */
    howManyDaysFit () {
      const stroke = parseFloat(this.root.style('calendar-row-rect')['border-width']);
      const additionalSpace = stroke * 2 + 2;
      let fullWidth = this.root.state.width;
      let formatNames = Object.keys(this.root.state.calendar.day.format);
      for (let days = this.root.state.times.steps.length; days > 1; days = Math.ceil(days / 2)) {
        for (let formatName of formatNames) {
          if (
            (this.root.state.calendar.day.maxWidths[formatName] + additionalSpace) * days <= fullWidth && days > 1) {
            return {
              count: days,
              type: formatName
            };
          }
        }
      }
      return {
        count: 0,
        type: ""
      };
    },

    /**
     * How many months will fit?
     *
     * @returns {object}
     */
    howManyMonthsFit () {
      const stroke = parseFloat(this.root.style('calendar-row-rect')['border-width']);
      const additionalSpace = stroke * 2 + 2;
      let fullWidth = this.root.state.width;
      let formatNames = Object.keys(this.root.state.calendar.month.format);
      let currentMonth = dayjs(this.root.state.times.firstDate);
      let previousMonth = currentMonth.clone();
      const lastTime = this.root.state.times.lastTime;
      let monthsCount = 1;
      while (currentMonth.valueOf() <= lastTime) {
        currentMonth = currentMonth.add(1, "day");
        if (previousMonth.month() !== currentMonth.month()) {
          monthsCount++;
        }
        previousMonth = currentMonth.clone();
      }
      for (let months = monthsCount; months > 1; months = Math.ceil(months / 2)) {
        for (let formatName of formatNames) {
          if ((this.root.state.calendar.month.maxWidths[formatName] + additionalSpace) * months <= fullWidth && months > 1) {
            return {
              count: months,
              type: formatName
            };
          }
        }
      }
      return {
        count: 1,
        type: formatNames[0]
      };
    },

    /**
     * Get hour text style
     *
     * @returns {string}
     */
    hourTextStyle () {
      return ("font-family:" + this.root.state.calendar.hour.fontFamily + ";font-size:" + this.root.state.calendar.hour.fontSize);
    },

    /**
     * Get text style
     *
     * @returns {string}
     */
    dayTextStyle () {
      return ("font-family:" + this.root.state.calendar.day.fontFamily + ";font-size:" + this.root.state.calendar.day.fontSize);
    },

    /**
     * Generate hours
     *
     * @returns {array}
     */
    generateHours () {
      let hours = [];
      if (!this.root.state.calendar.hour.display) {
        return this.root.state.calendar.hours = hours;
      }
      for (let hourIndex = 0, len = this.root.state.times.steps.length; hourIndex < len; hourIndex++) {
        const hoursCount = this.howManyHoursFit(hourIndex);
        const hourStep = 24 / hoursCount.count;
        const hourWidthPx = this.root.state.times.steps[hourIndex].width.px / hoursCount.count;
        for (let i = 0, len = hoursCount.count; i < len; i++) {
          const date = dayjs(this.root.state.times.steps[hourIndex].date).add(i * hourStep, "hour");
          let textWidth = 0;
          if (typeof this.root.state.calendar.hour.widths[hourIndex] !== 'undefined') {
            textWidth = this.root.state.calendar.hour.widths[hourIndex][hoursCount.type];
          }
          let x = this.root.state.times.steps[hourIndex].offset.px + hourWidthPx * i;
          hours.push({
            index: hourIndex,
            key: this.root.state.times.steps[hourIndex].date.valueOf() + "h" + i,
            x,
            y: this.root.state.calendar.day.height + this.root.state.calendar.month.height,
            width: hourWidthPx,
            textWidth,
            height: this.root.state.calendar.hour.height,
            label: this.root.state.calendar.hour.format[hoursCount.type](date)
          });
        }
      }
      return this.root.state.calendar.hours = hours;
    },

    /**
     * Generate days
     *
     * @returns {array}
     */
    generateDays () {
      let days = [];
      if (!this.root.state.calendar.day.display) {
        return this.root.state.calendar.days = days;
      }
      const daysCount = this.howManyDaysFit();
      const dayStep = Math.ceil(this.root.state.times.steps.length / daysCount.count);
      for (let dayIndex = 0, len = this.root.state.times.steps.length; dayIndex < len; dayIndex += dayStep) {
        let dayWidthPx = 0;
        // day could be shorter (daylight saving time) so join widths and divide
        for (let currentStep = 0; currentStep < dayStep; currentStep++) {
          if (typeof this.root.state.times.steps[dayIndex + currentStep] !== "undefined") {
            dayWidthPx += this.root.state.times.steps[dayIndex + currentStep].width.px;
          }
        }
        const date = dayjs(this.root.state.times.steps[dayIndex].date);
        let textWidth = 0;
        if (typeof this.root.state.calendar.day.widths[dayIndex] !== 'undefined') {
          textWidth = this.root.state.calendar.day.widths[dayIndex][daysCount.type];
        }
        let x = this.root.state.times.steps[dayIndex].offset.px;
        days.push({
          index: dayIndex,
          key: this.root.state.times.steps[dayIndex].date.valueOf() + "d",
          x,
          y: this.root.state.calendar.month.height,
          width: dayWidthPx,
          textWidth,
          height: this.root.state.calendar.day.height,
          label: this.root.state.calendar.day.format[daysCount.type](date)
        });
      }
      return this.root.state.calendar.days = days;
    },

    /**
     * Generate months
     *
     * @returns {array}
     */
    generateMonths () {
      let months = [];
      if (!this.root.state.calendar.month.display) {
        return this.root.state.calendar.months = months;
      }
      const monthsCount = this.howManyMonthsFit();
      let formatNames = Object.keys(this.root.state.calendar.month.format);
      let currentDate = dayjs(this.root.state.times.firstDate);
      for (let monthIndex = 0; monthIndex < monthsCount.count; monthIndex++) {
        let monthWidth = 0;
        let monthOffset = Number.MAX_SAFE_INTEGER;
        let finalDate = dayjs(currentDate).add(1, "month").startOf("month");
        if (finalDate.valueOf() > this.root.state.times.lastDate.valueOf()) {
          finalDate = dayjs(this.root.state.times.lastDate);
        }
        // we must find first and last step to get the offsets / widths
        for (let step = 0, len = this.root.state.times.steps.length; step < len; step++) {
          let currentStep = this.root.state.times.steps[step];
          if (currentStep.date.valueOf() >= currentDate.valueOf() && currentStep.date.valueOf() < finalDate.valueOf()) {
            monthWidth += currentStep.width.px;
            if (currentStep.offset.px < monthOffset) {
              monthOffset = currentStep.offset.px;
            }
          }
        }
        let label = "";
        let choosenFormatName;
        for (let formatName of formatNames) {
          if (this.root.state.calendar.month.maxWidths[formatName] + 2 <= monthWidth) {
            label = this.root.state.calendar.month.format[formatName](currentDate);
            choosenFormatName = formatName;
          }
        }
        let textWidth = 0;
        if (typeof this.root.state.calendar.month.widths[monthIndex] !== 'undefined') {
          textWidth = this.root.state.calendar.month.widths[monthIndex][choosenFormatName];
        }
        let x = monthOffset;
        months.push({
          index: monthIndex,
          key: monthIndex + "m",
          x,
          y: 0,
          width: monthWidth,
          textWidth,
          choosenFormatName,
          height: this.root.state.calendar.month.height,
          label
        });
        currentDate = currentDate.add(1, "month").startOf("month");
        if (currentDate.valueOf() > this.root.state.times.lastDate.valueOf()) {
          currentDate = dayjs(this.root.state.times.lastDate);
        }
      }
      return this.root.state.calendar.months = months;
    },

    /**
     * Regenerate dates
     */
    regenerate () {
      this.generateHours();
      this.generateDays();
      this.generateMonths();
      this.root.calculateCalendarDimensions();
    }

  },

  computed: {

    /**
     * Get width
     *
     * @returns {number}
     */
    getWidth () {
      let width = this.root.state.width;
      return width;
    },

    /**
     * Get month style
     *
     * @returns {object}
     */
    monthsStyle () {
      return this.root.mergeDeep({}, this.root.state.calendar.styles.row, this.root.state.calendar.month.style);
    },

    /**
     * Get day style
     *
     * @returns {object}
     */
    daysStyle () {
      return this.root.mergeDeep({}, this.root.state.calendar.styles.row, this.root.state.calendar.day.style);
    },

    /**
     * Get hour styke
     *
     * @returns {object}
     */
    hoursStyle () {
      return this.root.mergeDeep({}, this.root.state.calendar.styles.row, this.root.state.calendar.hour.style);
    },

    /**
     * Get visible days
     *
     * @returns {array}
     */
    getDays () {
      return this.days.filter(day => this.root.isInsideViewPort(day.x, day.width));
    },

    /**
     * Get visible hours
     *
     * @returns {array}
     */
    getHours () {
      return this.hours.filter(hour => this.root.isInsideViewPort(hour.x, hour.width));
    },

    /**
     * Get visible months
     *
     * @returns {array}
     */
    getMonths () {
      return this.months.filter(month => this.root.isInsideViewPort(month.x, month.width));
    },
  }
};
</script>
