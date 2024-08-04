<template>
  <div class="timepicker">
    <div class="input-wrapper">
      <input
          type="text"
          :value="internalValue"
          @focus="onfocus"
          @blur="onBlur"
          @keydown="handleKeyDown"
          @input="onInput"
          :placeholder="placeholder"
          ref="input"
      >
      <span class="clock-icon">
        <font-awesome-icon icon="clock" @click="onClockClick"/>
      </span>
    </div>
    <div v-if="showDropdown" class="dropdown">
      <div class="hours">
        <div
            v-for="hour in hourOptions"
            :key="`hour-${hour}`"
            @mousedown.prevent="selectHour(hour)"
            :class="{ selected: hour === currentHour }"
        >
          {{ formatHourDisplay(hour) }}
        </div>
      </div>
      <div class="minutes">
        <div
            v-for="minute in minuteOptions"
            :key="`minute-${minute}`"
            @mousedown.prevent="selectMinute(minute)"
            :class="{ selected: minute === currentMinute }"
        >
          {{ padZero(minute) }}
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  props: {
    value: {
      type: String,
      default: ''
    },
    minuteInterval: {
      type: Number,
      default: 1,
      validator: (value) => value > 0 && value <= 60 && 60 % value === 0
    },
    startTime: {
      type: String,
      default: '00:00'
    },
    endTime: {
      type: String,
      default: '23:59'
    }
  },
  data() {
    return {
      showDropdown: false,
      internalValue: this.value,
    }
  },
  computed: {
    hourOptions() {
      const [startHour, endHour] = [this.startTime, this.endTime].map(time => this.parseTime(time).totalMinutes / 60)
      return Array.from({length: Math.floor(endHour - startHour) + 1}, (_, i) => Math.floor(startHour) + i)
    },
    minuteOptions() {
      const options = []
      const {hour: currentHour} = this.parseTime(this.internalValue)
      const {totalMinutes: startMinutes} = this.parseTime(this.startTime)
      const {totalMinutes: endMinutes} = this.parseTime(this.endTime)

      for (let i = 0; i < 60; i += this.minuteInterval) {
        const currentTotalMinutes = currentHour * 60 + i
        if (currentTotalMinutes < startMinutes) continue
        if (currentTotalMinutes > endMinutes) break
        options.push(i)
      }
      return options
    },
    placeholder() {
      return `HH:MM`
    },
    // displayValue() {
    //   return this.internalValue || ''
    // },
    currentHour() {
      return this.parseTime(this.internalValue).hour
    },
    currentMinute() {
      return this.parseTime(this.internalValue).minute
    },
    minHour() {
      return this.parseTime(this.startTime).hour
    },
    maxHour() {
      return this.parseTime(this.endTime).hour
    },
    minMinute() {
      return this.parseTime(this.startTime).minute
    },
    maxMinute() {
      return this.parseTime(this.endTime).minute
    }
  },
  watch: {
    value(newValue) {
      this.internalValue = newValue
    }
  },
  methods: {
    padZero(num) {
      return num.toString().padStart(2, '0')
    },
    parseTime(timeString) {
      const [hours, minutes] = (timeString || '00:00').split(':').map(Number)
      return {
        hour: hours,
        minute: minutes,
        totalMinutes: hours * 60 + minutes
      }
    },
    formatTime(hours, minutes) {
      return `${this.padZero(hours)}:${this.padZero(minutes)}`
    },
    updateValue(value) {
      const validated = this.validateInput(value)
      this.internalValue = validated
      this.$emit('input', validated)
    },
    validateInput(value) {
      const regex = /^([0-9]{1,2}):([0-5][0-9])$/
      if (regex.test(value)) {
        const {totalMinutes: inputMinutes} = this.parseTime(value)
        const {totalMinutes: startMinutes} = this.parseTime(this.startTime)
        const {totalMinutes: endMinutes} = this.parseTime(this.endTime)

        if (inputMinutes >= startMinutes && inputMinutes <= endMinutes) {
          const {hour, minute} = this.parseTime(value)
          const roundedMinute = this.roundToNearestInterval(minute)
          return this.formatTime(hour, roundedMinute)
        } else {
          if (inputMinutes < startMinutes) {
            return this.startTime
          } else {
            return this.endTime
          }
        }
      }
      return this.internalValue;
    },
    roundToNearestInterval(minute) {
      return Math.round(minute / this.minuteInterval) * this.minuteInterval
    },
    onBlur() {
      setTimeout(() => {
        this.showDropdown = false
        this.updateValue(this.internalValue)
      }, 200)
    },
    formatHourDisplay(hour) {
      return this.padZero(hour)
    },
    selectHour(hour) {
      const {minute} = this.parseTime(this.internalValue);
      this.internalValue = this.formatTime(hour, minute || 0);
    },
    selectMinute(minute) {
      const {hour} = this.parseTime(this.internalValue);
      this.internalValue = this.formatTime(hour || 0, minute);
    },
    handleKeyDown(event) {
      if (event.key === 'ArrowUp' || event.key === 'ArrowDown') {
        event.preventDefault();
        const increment = event.key === 'ArrowUp' ? 1 : -1;
        const cursorPosition = event.target.selectionStart;
        const [currentHour, currentMinute] = this.internalValue.split(':').map(Number);

        if (cursorPosition <= 2) {
          this.adjustHour(currentHour, increment, cursorPosition);
        } else {
          this.adjustMinute(currentMinute, increment, cursorPosition);
        }
      }
    },

    adjustHour(currentHour, increment, cursorPosition) {
      const {totalMinutes: startMinutes} = this.parseTime(this.startTime);
      const {totalMinutes: endMinutes} = this.parseTime(this.endTime);
      let newHour = currentHour + increment;
      let newTotalMinutes = newHour * 60 + this.currentMinute;

      if (newTotalMinutes < startMinutes) {
        newHour = Math.floor(endMinutes / 60);
      } else if (newTotalMinutes > endMinutes) {
        newHour = Math.floor(startMinutes / 60);
      }

      this.internalValue = this.formatTime(newHour, this.currentMinute);
      this.$nextTick(() => {
        this.$el.querySelector('input').setSelectionRange(cursorPosition, cursorPosition);
      });
    },

    adjustMinute(currentMinute, increment, cursorPosition) {
      const {totalMinutes: startMinutes} = this.parseTime(this.startTime);
      const {totalMinutes: endMinutes} = this.parseTime(this.endTime);
      let newMinute = currentMinute + increment * this.minuteInterval;
      let newHour = this.currentHour;
      let newTotalMinutes = newHour * 60 + newMinute;

      if (newTotalMinutes < startMinutes) {
        [newHour, newMinute] = this.parseTime(this.endTime);
      } else if (newTotalMinutes > endMinutes) {
        [newHour, newMinute] = this.parseTime(this.startTime);
      } else if (newMinute >= 60) {
        newHour++;
        newMinute -= 60;
      } else if (newMinute < 0) {
        newHour--;
        newMinute += 60;
      }

      this.internalValue = this.formatTime(newHour, newMinute);
      this.$nextTick(() => {
        this.$el.querySelector('input').setSelectionRange(cursorPosition, cursorPosition);
      });
    },
    onClockClick(event) {
      event.preventDefault();
      this.toggleDropdown();
    },
    toggleDropdown() {
      this.showDropdown = !this.showDropdown;
      if (this.showDropdown) {
        this.$nextTick(() => {
          this.$refs.input.focus();
        });
      } else {
        this.$nextTick(() => {
          this.$refs.input.blur();
        });
      }
    },
    onInput(event) {
      const input = event.target;
      const newValue = input.value;
      const cursorPosition = input.selectionStart;
      const currentChar = newValue.slice(cursorPosition - 1, cursorPosition);
      this.internalValue = this.replaceCharAt(this.internalValue, cursorPosition - 1, currentChar);
      this.$nextTick(() => {
        event.target.value = this.internalValue;
        if (cursorPosition === 2) {
          this.$el.querySelector('input').setSelectionRange(3, 3);
        } else {
          this.$el.querySelector('input').setSelectionRange(cursorPosition, cursorPosition);
        }
      });
    },
    onfocus(event) {
      this.showDropdown = true;
      this.$nextTick(() => {
        event.target.setSelectionRange(0, 0);
      });
    },
    replaceCharAt(str, index, replacement) {
      const [startHours, startMinutes] = this.startTime.split(':').map(Number);
      const [endHours, endMinutes] = this.endTime.split(':').map(Number);

      let newStr = str.substring(0, index) + replacement + str.substring(index + 1);
      let [newHours, newMinutes] = newStr.split(':').map(Number);

      // 時の補正
      if (index < 2) {
        if (newHours < startHours) {
          newHours = startHours;
          newMinutes = startMinutes;
        } else if (newHours > endHours) {
          newHours = endHours;
          newMinutes = endMinutes;
        } else if (newHours === endHours && newMinutes > endMinutes) {
          newMinutes = endMinutes;
        }
      }
      // 分の補正
      else {
        if (newHours < startHours || (newHours === startHours && newMinutes < startMinutes)) {
          newHours = startHours;
          newMinutes = startMinutes;
        } else if (newHours > endHours || (newHours === endHours && newMinutes > endMinutes)) {
          newHours = endHours;
          newMinutes = endMinutes;
        } else {
          // 分が 0-59 の範囲内に収まるように補正
          newMinutes = Math.max(0, Math.min(59, newMinutes));
        }
      }

      // 分が this.minuteInterval の倍数になるように調整
      newMinutes = Math.round(newMinutes / this.minuteInterval) * this.minuteInterval;

      // 調整後の分が 60 以上になった場合の処理
      if (newMinutes >= 60) {
        newHours++;
        newMinutes = 0;
      }

      // 最終的な時刻が範囲外の場合、範囲内に収める
      if (newHours > endHours || (newHours === endHours && newMinutes > endMinutes)) {
        newHours = endHours;
        newMinutes = endMinutes;
      }

      return `${newHours.toString().padStart(2, '0')}:${newMinutes.toString().padStart(2, '0')}`;
    },
  }
}
</script>

<style scoped>
.timepicker {
  position: relative;
  width: 120px;
}

.input-wrapper {
  position: relative;
  display: inline-block;
  width: 100%;
}

.timepicker input {
  width: 5rem;
  padding: 0.5rem;
  border: 1px solid #ccc;
  border-radius: 0.5rem;
}

.clock-icon {
  position: absolute;
  right: 1.4rem;
  top: 53%;
  transform: translateY(-50%);
  color: #888;
  cursor: pointer;
}

.dropdown {
  position: absolute;
  left: 12px;
  display: flex;
  border: 1px solid #ccc;
  background-color: white;
  z-index: 1000;
}

.hours, .minutes {
  max-height: calc(100vh / 8);
  overflow-y: auto;
  width: 48px;

  /* スクロールバー全体のスタイル */
  & .custom-scrollbar::-webkit-scrollbar {
    width: 8px; /* スクロールバーの幅 */
    height: 8px; /* スクロールバーの高さ（水平スクロール） */
  }

  /* スクロールバーのトラック（背景） */
  & .custom-scrollbar::-webkit-scrollbar-track {
    background: transparent; /* 背景を透明に */
  }

  /* スクロールバーのつまみ（スライダー） */
  & .custom-scrollbar::-webkit-scrollbar-thumb {
    background: rgba(0, 0, 0, 0.5); /* 半透明の黒 */
    border-radius: 4px; /* 角を丸くする */
  }

  /* スクロールバーつまみのホバー時のスタイル */
  & .custom-scrollbar::-webkit-scrollbar-thumb:hover {
    background: rgba(0, 0, 0, 0.7); /* ホバー時に少し濃く */
  }

  /* スクロールバーのコーナー（水平・垂直スクロールが交差する部分） */
  & .custom-scrollbar::-webkit-scrollbar-corner {
    background: transparent; /* 背景を透明に */
  }
}


.hours div, .minutes div {
  padding: 5px;
  cursor: pointer;
  text-align: center;
}

.hours div:hover, .minutes div:hover {
  background-color: #f0f0f0;
}

.hours div.selected, .minutes div.selected {
  background-color: #007bff;
  color: white;
}
</style>