<template>
  <div class="calendar-container" v-if="showCalendar">
    <div class="calendar-bg" @click="hideCalendar({emitStatu: true})"></div>
    <div class="calendar-body"
      @touchstart="bindTouchStart"
      @touchend="bindTouchEnd"
    >
      <div class="calendar-title">
        <span class="calendar-title-cancle" @click="hideCalendar({emitStatu: true})">取消</span>
        <div class="calendar-title-choose">
          <span class="calendar-title-icon calendar-title-icon-left" @click="preMonth"></span>
          <span class="calendar-title-show"><span>{{year}} - {{month}}</span></span>
          <span class="calendar-title-icon calendar-title-icon-right" @click="nextMonth"></span>
        </div>
        <span class="calendar-title-confirm" @click="resetCalendar">重置</span>
      </div>
      <div class="calendar-week">
        <span v-for="(item, index) in week" :key="index" class="calendar-week-item"
          :class="{'calendar-week-item-holiday': index === 5 || index === 6}"
        >{{item}}</span>
      </div>
      <div class="calendar-day">
        <div v-for="(item, index) in dayData" :key="index" class="calendar-day-row">
          <div v-for="(day, dindex) in item" :key="dindex" class="calendar-day-item"
            :class="{'week-7': day!== '' && (dindex === 5 || dindex === 6),
              'calendar-disabled': day.disabled
            } "
            :id="day.id"
            @click="chooseDate(day)"
          >
          <span class="calendar-day-item-num"
            v-if="day !== ''"
            :class="{
              'calendar-checked-default':chooseToday === day.id,
              'calendar-checked': chooseDay === day.id}"
          >{{day.number}}</span>
          <span class="calendar-day-item-holiday" v-if="day.holiday">休</span>
          <span class="calendar-day-item-text" v-if="day !== ''">{{day.text?day.text:''}}</span>
          <span class="calendar-day-item-flag" :class="{'calendar-day-item-flag-checked': day.haveFlag}" v-if="day !== ''"></span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
import Calendar from '@/plugs/chinese-calendar';
import HolidayData from '@/plugs/holiday.data';

export default {
  /***
   * 说明：日历组件，适用于vue,mpvue
   * 需求：node-sass, sass-loader (配置方式请百度)
   * 功能：
   * 1.支持农历和节假日（农历1900-2100）
   * 2.支持左右滑动切换，和点击箭头切换
   * 3.点击‘重置’按钮，回到当前日期
   * 4.点击‘取消’按钮和灰色背景隐藏组件
   * 5.支持特殊日期标记圆点
   * 6.假期显示，需要在plugs/holida.data.js手动添加数据
   *
   * 参数说明：
   * :params.id  可以在同一个页面使用多个此组件，返回日期的时候会携带id
   * :params.chooseDayText  选择的日期 '2018-1-2'
   * :params.minDate  最小选择日期 '2018-10-11'
   * :params.maxDate  最大选择日期 '2019-2-10'
   * :activityData  有活动的日期显示flag圆点 {year:{ month:[]}} 月和日小于10不可以有0，01 =》1
   *
   * 函数说明：
   * @chooseDate  选择日期之后执行的函数 { data: ‘2018-11-28’, y: '2018', m: '11', d: '28', id: this.params.id};
   * @hideChooseDate  日历隐藏时执行函数
   * @changeDate  切换月份的时候执行函数，返回切换后的年月日。 { year: ‘2018’, month: ‘11’, day: ‘28’ };
   *
   * 问题解决：
   * 1.chinese-calendar.js 样式报错。
   * 解决： build/webpack.base.conf.js   rule > eslint-loader > 添加 exclude: [resolve('src/plugs')]
   *
   * 2. 默认是px单位，可以修改 $rpx: 1rpx; 变量改为 rpx
   **/
  name: 'comCalendar',
  props: ['params', 'activityData'],
  data () {
    return {
      todayObj: {},
      chooseDay: '', // 选择的日期
      chooseToday: '',
      year: '',
      month: '',
      day: '', // 今日
      week: ['一', '二', '三', '四', '五', '六', '日'],
      dayData: [],
      limitMinDate: null,
      limitMaxDate: null,
      showCalendar: true,
      flagArr: null,
      checkedMonthFlagArr: null,

      touchObj: {},
      touchLimit: 30 // px
    };
  },
  watch: {
    activityData: {
      immediate: true,
      handler () {
        if (this.year && this.month) {
          this.setFlagArr();
        }
      }
    }
  },
  mounted () {
    let date = new Date();

    this.todayObj = {
      year: date.getFullYear(),
      month: date.getMonth() + 1,
      day: date.getDate()
    };

    this.chooseToday = this.todayObj.year + '-' + this.todayObj.month + '-' + this.todayObj.day;

    if (this.params && this.params.minDate) {
      let splitArr = this.params.minDate.split('-');
      this.limitMinDate = {
        y: splitArr[0] * 1,
        m: splitArr[1] * 1,
        d: splitArr[2] * 1
      };
    }

    if (this.params && this.params.maxDate) {
      let splitArr = this.params.maxDate.split('-');
      this.limitMaxDate = {
        y: splitArr[0] * 1,
        m: splitArr[1] * 1,
        d: splitArr[2] * 1
      };
    }

    if (this.params && this.params.chooseDayText) {
      this.chooseDay = this.params.chooseDayText;
      let splitArr = this.params.chooseDayText.split('-');
      let request = {
        year: splitArr[0] * 1,
        month: splitArr[1] * 1,
        day: splitArr[2] * 1
      };

      this.setDateParams(request);
    } else {
      this.setDateParams(this.todayObj);
    }
  },
  methods: {
    setDateParams (obj) {
      this.year = obj.year;
      this.month = obj.month;
      this.day = obj.day;

      this.setFlagArr();
    },

    setFlagArr () {
      // 判断是否有需要特殊显示的日期
      if (this.activityData && this.activityData[parseInt(this.year)] && this.activityData[parseInt(this.year)][parseInt(this.month)]) {
        this.flagArr = this.activityData[parseInt(this.year)][parseInt(this.month)];
      }

      this.getShowDayList();
    },

    getShowDayList () {
      this.dayData = [];
      let row = [];
      for (let i = 1; i < 42; i++) { // 7 * 5line
        let day = new Date(this.year, this.month - 1, i);
        let weekDay = day.getDay();
        let month = day.getMonth() + 1;
        let monthDay = day.getDate();
        let year = day.getFullYear();
        let str = `${year}-${month}-${monthDay}`;

        let obj = {
          text: null,
          id: str,
          number: null,
          disabled: false,
          haveFlag: false
        };

        // 判断是否突出显示当天日期，例如当天有活动
        if (this.flagArr) {
          for (let i = 0; i < this.flagArr.length; i++) {
            if (parseInt(this.flagArr[i].day) === monthDay) {
              obj.haveFlag = true;
              break;
            }
          }
        }

        // 判断是否为假期
        if (HolidayData[year] && HolidayData[year][month]) {
          let holidayArr = HolidayData[year][month];
          if (holidayArr.length > 0) {
            for (let i = 0; i < holidayArr.length; i++) {
              if (holidayArr[i] === monthDay) {
                obj.holiday = true;
                break;
              }
            }
          }
        }

        // 假期，农历显示
        let calendarTextObj = Calendar.solar2lunar(year, month, monthDay);
        if (calendarTextObj.festival.length > 0) {
          obj.text = calendarTextObj.festival[0];
        } else
        if (calendarTextObj.IMonthCn && calendarTextObj.IDayCn) {
          if (calendarTextObj.IDayCn === '初一') {
            obj.text = calendarTextObj.IMonthCn;
          } else {
            obj.text = calendarTextObj.IDayCn;
          }
        }

        // 限制选取最小日期
        if (this.limitMinDate && (year < this.limitMinDate.y ||
          (year === this.limitMinDate.y && month < this.limitMinDate.m) ||
          (year === this.limitMinDate.y && month === this.limitMinDate.m && monthDay < this.limitMinDate.d))
        ) {
          obj.disabled = true;
        }

        // 限制选取最大日期
        if (this.limitMaxDate && (year > this.limitMaxDate.y ||
          (year === this.limitMaxDate.y && month > this.limitMaxDate.m) ||
          (year === this.limitMaxDate.y && month === this.limitMaxDate.m && monthDay > this.limitMaxDate.d))
        ) {
          obj.disabled = true;
        }

        // 构造本月的日历数据
        if (month === this.month) {
          if (i === 1) {
            // 判断第一天的日期，
            // @d 用于控制第一天显示的星期 （d = 0 开头为星期日 / d = 1 开头为星期一）
            for (let d = 1; d < weekDay; d++) {
              row.push('');
            }

            obj.number = monthDay;
            row.push(obj);
          } else {
            obj.number = monthDay;
            row.push(obj);
          }
        } else {
          row.push('');
        }

        if (row.length % 7 === 0) {
          if (i > 35 && row[0] === '') {
            row = [];
            return;
          }
          this.dayData.push(row);
          row = [];
        }
      };
    },

    preMonth () {
      this.setMonth(parseInt(this.month) - 1);
    },

    nextMonth () {
      this.setMonth(parseInt(this.month + 1));
    },

    setMonth (month) {
      let setDate = new Date(this.year, month - 1, this.day);
      let params = {
        year: setDate.getFullYear(),
        month: setDate.getMonth() + 1,
        day: setDate.getDate()
      };

      this.flagArr = null;
      this.setDateParams(params);
      this.$emit('changeDate', params);
    },

    chooseDate (params) {
      if (!params.disabled) {
        this.chooseDay = params.id;

        let arr = params.id.split('-');
        let result = {
          data: params.id,
          y: arr[0],
          m: arr[1],
          d: arr[2],
          id: this.params.id
        };

        this.$emit('chooseDate', result);
        this.hideCalendar({emitStatu: false});
      }
    },

    resetCalendar () {
      this.chooseDay = null;

      let result = {
        data: undefined,
        y: null,
        m: null,
        d: null,
        id: this.params.id
      };

      this.$emit('chooseDate', result);
      this.hideCalendar({emitStatu: false});
    },

    hideCalendar (e) {
      this.showCalendar = false;
      if (e.emitStatu) {
        this.emitGetActivityParams();
      }
      this.$emit('hideChooseDate');
    },

    emitGetActivityParams () {
      let params = null;
      if (this.params && this.params.chooseDayText) {
        let splitArr = this.params.chooseDayText.split('-');
        params = {
          year: splitArr[0],
          month: splitArr[1],
          day: splitArr[2]
        };
      } else {
        params = {
          year: this.todayObj.year,
          month: this.todayObj.month,
          day: this.todayObj.day
        };
      }

      this.$emit('changeDate', params);
    },

    bindTouchStart (e) {
      if (e.mp) {
        this.touchObj['start'] = e.mp.changedTouches[0].clientX;
      } else {
        this.touchObj['start'] = e.changedTouches[0].clientX;
      }
    },

    bindTouchEnd (e) {
      if (e.mp) {
        this.touchObj['end'] = e.mp.changedTouches[0].clientX;
      } else {
        this.touchObj['end'] = e.changedTouches[0].clientX;
      }

      if (this.touchObj.end - this.touchObj.start > this.touchLimit) {
        this.preMonth();
      } else
      if (this.touchObj.end - this.touchObj.start < -this.touchLimit) {
        this.nextMonth();
      }
    }
  }
};
</script>
<style lang="scss" scoped>
  $rpx: 0.5px;
  $topic-color: #32DA31;
  $orange-color: #FFB62E;
  $weekday-color: #fff3dd;

  .calendar-container {
    position: fixed;
    bottom: 0;
    left: 0;
    width: 100%;
    height: 100vh;
    overflow: hidden;
    z-index: 999;

    .calendar-bg {
      width: 100%;
      height: 100%;
      background-color: rgba(0, 0, 0, 0.5);
    }

    .calendar-body {
      position: absolute;
      bottom: 0;
      left: 0;
      width: 100%;
      padding: 30*$rpx 0;
      background-color: #ffffff;

      .calendar-title {
        display: flex;
        justify-content: space-between;
        align-items: center;
        flex-flow: row nowrap;
        margin: 0 5%;

        .calendar-title-confirm {
          color: $topic-color;
          flex-shrink: 0;
        }

        .calendar-title-cancle {
          color: #515151;
          flex-shrink: 0;
        }

        .calendar-title-choose {
          display: flex;
          justify-content: center;
          align-items: center;
          flex-flow: row nowrap;

          .calendar-title-icon {
            height: 40*$rpx;
            width: 40*$rpx;
            padding: 0 50*$rpx;
            flex-shrink: 0;
          }

          .calendar-title-icon-left {
            background: url('../assets/calendar_arrow_left.png') no-repeat 50% 50%;
            background-size: contain;
          }

          .calendar-title-icon-right {
            background: url('../assets/calendar_arrow_right.png') no-repeat 50% 50%;
            background-size: contain;
          }


          .calendar-title-show {
            flex-shrink: 0;
            span {
              font-weight: bold;
              margin-right: 10*$rpx;
            }
          }
        }
      }

      .calendar-week {
        display: flex;
        justify-content: space-between;
        align-items: center;
        flex-flow: row nowrap;

        margin-top: 30*$rpx;
        padding-top: 20*$rpx;
        border-top: 1*$rpx solid #dcdcdc;

        .calendar-week-item {
          text-align: center;
          flex-basis: calc(100% / 7);
          padding: 10*$rpx 0;
          font-weight: bold;
        }

        .calendar-week-item-holiday {
          color: #9d9d9d;
        }
      }

      .calendar-day {
        .calendar-day-row {
          display: flex;
          justify-content: space-between;
          align-items: flex-start;
          flex-flow: row nowrap;

          .calendar-day-item {
            position: relative;
            text-align: center;
            flex-basis: calc(100% / 7);
            padding: 16*$rpx 0;
            display: flex;
            justify-content: center;
            align-items: center;
            flex-flow: column nowrap;

            .calendar-day-item-num {
              width: 48*$rpx;
              height: 48*$rpx;
              text-align: center;
              line-height: 48*$rpx;
              border-radius: 100%;
              font-weight: bold;
            }

            .calendar-day-item-holiday {
              position: absolute;
              right: 8*$rpx;
              top: 40%;
              transform: translateY(-40%);
              font-size: 10px;
              color: $orange-color;
            }

            .calendar-day-item-text  {
              font-size: 10px;
              line-height:normal;
              height: 30*$rpx;
            }

            .calendar-day-item-flag {
              width: 10*$rpx;
              height: 10*$rpx;
              border-radius: 100%;
            }

            .calendar-day-item-flag-checked {
              background-color: $orange-color;
            }
          }

          .week-7 {
            background-color: $weekday-color;
          }

          .calendar-checked-default {
            background-color: $topic-color;
            color: #ffffff;
          }

          .calendar-checked {
            background-color: $orange-color;
            color: #ffffff;
          }

          .calendar-disabled {
            opacity: 0.7;
          }
        }
      }
    }
  }
</style>
