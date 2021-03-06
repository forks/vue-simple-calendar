<template>
	<div class="calendar"
		:class="[
				'locale-' + languageCode,
				'locale-' + locale,
				'y' + showDate.getFullYear(),
				'm' + paddedMonth(showDate),
				{
					past: beginningOfMonth(today) < showDate,
					future: beginningOfMonth(today) > showDate
				}
			]">
		<div class="header">
			<div class="previousYear"><button @click="onClickPreviousYear" :disabled="disablePast && (today > endOfPreviousYear(showDate))"></button></div>
			<div class="previousMonth"><button @click="onClickPreviousMonth" :disabled="disablePast && (today > endOfPreviousMonth(showDate))"></button></div>
			<div class="thisMonth">
				<div class="monthLabel">
					<span class="monthName">{{monthName(showDate)}}</span>
					<span class="yearNumber">{{showDate.getFullYear()}}</span>
				</div>
				<div v-if="!isSameMonth(today, showDate)" class="currentMonth"><button @click="onClickCurrentMonth"></button></div>
			</div>
			<div class="nextMonth"><button @click="onClickNextMonth" :disabled="disableFuture && (today < beginningOfNextMonth(showDate))"></button></div>
			<div class="nextYear"><button @click="onClickNextYear" :disabled="disableFuture && (today < beginningOfNextYear(showDate))"></button></div>
		</div>
		<div class="dayList">
			<div class="day" v-for="(w, index) in weekdayNames" :class="'dow'+index">{{w}}</div>
		</div>
		<div class="month">
			<div v-for="(weekStart, weekIndex) in weeks" class="week" :class="['week' + (weekIndex+1), 'ws' + isoYearMonthDay(weekStart)]">
				<div v-for="day in daysOfWeek(weekStart)" class="day" 
					@drop.prevent="onDrop(day, $event)"
					@dragover.prevent="onDragOver(day, $event)"
					@dragenter.prevent="onDragEnter(day, $event)"
					@dragleave.prevent="onDragLeave(day, $event)"
					:class="[
						'dow' + day.getDay(),
						'd' + isoYearMonthDay(day),
						'd' + isoMonthDay(day),
						'd' + paddedDay(day),
						'instance' + instanceOfMonth(day),
						{
							outsideOfMonth : day.getMonth() != showDate.getMonth(),
							today : isSameDate(day, today),
							past : isInPast(day),
							future : isInFuture(day),
							last: day.getMonth() !== addDays(day, 1).getMonth(),
							lastInstance: lastInstanceOfMonth(day),
						}
					]" @click="onClickDay(day)">
					<div class="content">
						<div class="date">{{day.getDate()}}</div>
					</div>
				</div>
				<div v-for="e in getWeekEvents(weekStart)"
					class="event"
					:draggable="enableDragDrop"
					@dragstart="onDragStart(e, $event)"
					@click.stop="onClickEvent(e)"
					:class="e.classes"
					:title="e.details.title"
					v-html="e.details.title"></div>
				</div>
		</div>
	</div>
</template>

<script>

export default {
	name: 'simple-calendar',

	data() {
		return {};
	},

	props: {
		events: {
			type: Array,
			default() { return []; },
		},
		showDate: {
			type: Date,
			default() {
				const d = new Date();
				d.setHours(0, 0, 0, 0);
				return d;
			},
		},
		monthNameFormat: {
			type: String,
			default() { return 'long'; },
		},
		weekdayNameFormat: {
			type: String,
			default() { return 'short'; },
		},
		locale: {
			type: String,
			default() {
				return (
					(navigator.languages && navigator.languages.length) ?
						navigator.languages[0]
						: navigator.language || navigator.browserLanguage
				).toLowerCase();
			},
		},
		disablePast: {
			type: Boolean,
			default() { return false; },
		},
		disableFuture: {
			type: Boolean,
			default() { return false; },
		},
		enableDragDrop: {
			default() { return false; },
		},
	},

	computed: {
		today() {
			const d = new Date();
			d.setHours(0, 0, 0, 0);
			return d;
		},
		languageCode() {
			return this.locale.substring(0, 2);
		},
		weeks() {
			// Returns an array of object representing the date of the beginning of each week
			// included in the view (which, by default, consists of an entire month).
			const firstDate = this.beginningOfCalendar(this.showDate);
			const lastDate = this.endOfCalendar(this.showDate);
			const numWeeks = Math.floor(this.dayDiff(firstDate, lastDate) / 7);
			const result = [];
			for (let x = 0; x < numWeeks; x++) {
				result.push(this.addDays(firstDate, x * 7));
			}
			return result;
		},
		weekdayNames() {
			if (typeof Intl === 'undefined') {
				return ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'];
			}
			const formatter = new Intl.DateTimeFormat(this.locale, { weekday: this.weekdayNameFormat });
			const names = [];
			for (let dayIndex = 0; dayIndex < 7; dayIndex++) {
				// 2017 starts on a Sunday, so use it to capture the locale's weekday names
				const sampleDate = new Date(2017, 0, dayIndex + 1, 0, 0, 0);
				names[dayIndex] = formatter.format(sampleDate);
			}
			return names;
		},
		allowLastMonthClick() {
			if (!this.disablePast) return true;
			const endOfLastMonth = this.addDays(this.beginningOfMonth(this.showDate), -1);
			return endOfLastMonth >= this.today;
		},
		allowNextMonthClick() {
			if (!this.disableFuture) return true;
			const beginningOfNextMonth = this.addDays(this.endOfMonth(this.showDate), 1);
			return beginningOfNextMonth <= this.today;
		},
	},

	methods: {

		addDays(d, days) {
			const d2 = new Date(d);
			d2.setDate(d.getDate() + days);
			return d2;
		},

		isSameDate(d1, d2) {
			// http://stackoverflow.com/questions/492994/compare-two-dates-with-javascript
			return d1.getTime() === d2.getTime();
		},

		isSameMonth(d1, d2) {
			return d1.getFullYear() === d2.getFullYear() && d1.getMonth() === d2.getMonth();
		},

		daysOfWeek(weekStart) {
			const result = [];
			for (let x = 0; x < 7; x++) result.push(this.addDays(weekStart, x));
			return result;
		},

		isInFuture(d) { return d > this.today; },

		isInPast(d) { return d < this.today; },

		instanceOfMonth: d => Math.ceil(d.getDate() / 7),

		isoYearMonthDay: d => d.toISOString().slice(0, 10),

		isoMonthDay: d => d.toISOString().slice(5, 10),

		isoYearMonth: d => d.toISOString().slice(0, 7),

		paddedMonth: d => ('0' + String(d.getMonth() + 1)).slice(-2),

		paddedDay: d => ('0' + String(d.getDate() + 1)).slice(-2),

		beginningOfMonth: d => new Date(d.getFullYear(), d.getMonth(), 1),

		endOfMonth: d => new Date(d.getFullYear(), d.getMonth() + 1, 0),

		endOfPreviousMonth: d => new Date(d.getFullYear(), d.getMonth(), 0),

		aYearAgo: d => new Date(d.getFullYear() - 1, d.getMonth(), 1),

		aYearFrom: d => new Date(d.getFullYear() + 1, d.getMonth(), 1),

		beginningOfPreviousMonth: d => new Date(d.getFullYear(), d.getMonth() - 1, 1),

		beginningOfNextMonth: d => new Date(d.getFullYear(), d.getMonth() + 1, 1),

		lastInstanceOfMonth(d) { return d.getMonth() !== this.addDays(d, 7).getMonth(); },

		beginningOfWeek(d) { return this.addDays(d, 0 - d.getDay()); },

		endOfWeek(d) { return this.addDays(d, 7 - d.getDay()); },

		beginningOfCalendar(d) { return this.beginningOfWeek(this.beginningOfMonth(d)); },

		endOfCalendar(d) { return this.endOfWeek(this.endOfMonth(d)); },

		// Number of days between two dates (times must be 0)
		dayDiff(d1, d2) { return (d2 - d1) / 86400000; },

		// Name of the given month (only used once)
		monthName(d) {
			// Use the user's locale if possible to obtain the name of the month
			if (typeof Intl === 'undefined') {
				return ['Jan', 'Feb', 'Mar', 'Apr', 'May',
					'June', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'][d.getMonth()];
			}
			const formatter = new Intl.DateTimeFormat(this.locale, { month: this.monthNameFormat });
			return formatter.format(d);
		},

		onClickDay(day) {
			if (this.disablePast && this.isInPast(day)) return;
			this.$emit('clickDay', day);
		},

		onClickEvent(e, day) {
			this.$emit('clickEvent', e.details, day);
		},

		onClickPreviousYear() { this.$emit('setShowDate', this.aYearAgo(this.showDate)); },
		onClickPreviousMonth() { this.$emit('setShowDate', this.beginningOfPreviousMonth(this.showDate)); },
		onClickNextMonth() { this.$emit('setShowDate', this.beginningOfNextMonth(this.showDate)); },
		onClickNextYear() { this.$emit('setShowDate', this.aYearFrom(this.showDate)); },
		onClickCurrentMonth() { this.$emit('setShowDate', this.beginningOfMonth(this.today)); },

		findAndSortEventsInWeek(weekStart) {
			// Return a list of events that CONTAIN the week starting on a day.
			// Sorted so the events that start earlier are always shown first.
			const events = this.events.filter(event =>
				event.startDate < this.addDays(weekStart, 7)
				&& event.endDate >= weekStart
			, this).sort((a, b) => {
				if (a.startDate < b.startDate) return -1;
				if (b.startDate < a.startDate) return 1;
				if (a.endDate > b.endDate) return -1;
				if (b.endDate > a.endDate) return 1;
				return a.id < b.id ? -1 : 1;
			});
			return events;
		},

		getWeekEvents(weekStart) {
			// Return a list of events that CONTAIN the week starting on a day.
			// Sorted so the events that start earlier are always shown first.
			const events = this.findAndSortEventsInWeek(weekStart);
			const results = [];
			const slots = [[], [], [], [], [], [], [], [], [], []];
			for (let i = 0; i < events.length; i++) {
				const e = events[i];
				const ep = { details: e, slot: 0 };
				const continued = e.startDate < weekStart;
				const startOffset = continued ? 0 : this.dayDiff(weekStart, e.startDate);
				const toBeContinued = this.dayDiff(weekStart, e.endDate) > 7;
				const span = Math.min(
					7 - startOffset,
					this.dayDiff(this.addDays(weekStart, startOffset), e.endDate) + 1);
				for (let d = 0; d < 7; d++) {
					if (d === startOffset) {
						for (let s = 0; s < 10; s++) {
							if (!slots[d][s]) {
								ep.slot = s;
								slots[d][s] = true;
								break;
							}
						}
					} else if (d < startOffset + span) {
						slots[d][ep.slot] = true;
					}
				}
				ep.classes = [
					`offset${startOffset}`,
					`span${span}`,
					`slot${ep.slot + 1}`,
					{
						continued,
						toBeContinued,
						hasUrl: e.url,
					},
				];
				if (e.classes) ep.classes = ep.classes.concat(e.classes);
				results.push(ep);
			}
			return results;
		},

		onDragStart(calendarEvent, windowEvent) {
			if (!this.enableDragDrop) return false;
			// Reason for using "Text":
			// http://stackoverflow.com/questions/26213011/html5-dragdrop-issue-in-internet-explorer-datatransfer-property-access-not-pos
			windowEvent.dataTransfer.setData('Text', calendarEvent.details.id);
			this.$emit('dragEventStart', calendarEvent.details.id, calendarEvent);
			return true;
		},

		handleEvent(windowEvent, bubbleEventName, bubbleParam) {
			if (!this.enableDragDrop) return false;
			const calendarEventId = windowEvent.dataTransfer.getData('Text');
			this.$emit(bubbleEventName, calendarEventId, bubbleParam);
			return true;
		},

		onDragOver(day, windowEvent) {
			this.handleEvent(windowEvent, 'dragEventDragOverDate', day);
		},

		onDragEnter(day, windowEvent) {
			if (!this.handleEvent(windowEvent, 'dragEventEnterDate', day)) return;
			windowEvent.target.classList.add('draghover');
		},

		onDragLeave(day, windowEvent) {
			if (!this.handleEvent(windowEvent, 'dragEventLeaveDate', day)) return;
			windowEvent.target.classList.remove('draghover');
		},

		onDrop(day, windowEvent) {
			if (!this.handleEvent(windowEvent, 'dropEventOnDate', day)) return;
			windowEvent.target.classList.remove('draghover');
		},

	},

};

</script>

<style type="text/css">

	.calendar,
	.calendar div {
		box-sizing: border-box;
	}

	.month,
	.header,
	.week,
	.dayList {
		display: flex;
		width: 100%;
		flex-wrap: wrap;
		flex-direction: row;
		justify-content: flex-start;
		align-items: stretch;
		align-content: flex-start;
		border-style: solid;
		border-color: #DDD;
	}

	.header {
		border-width: 0.05em 0.05em 0 0.05em;
		justify-content: space-between;
	}

	.header > div {
		margin: 0.3em;
	}

	.header .thisMonth {
		display: flex;
		flex-direction: row;
	}

	.header .monthLabel {
		padding: 0.25em;
		font-weight: bold;
	}

	.header button {
		padding: 0.4em 0.6em;
		background-color: transparent;
		border: 1px solid #e7e7e7;
		color: #808080;
		line-height: 1em;
		font-size: 1em;
	}

	.header .previousMonth button::after { content: "\25C0"; }
	.header .nextMonth button::after { content: "\25B6"; }
	.header .previousYear button::after { content: "\25C0\25C0"; }
	.header .nextYear button::after { content: "\25B6\25B6"; }

	.header .currentMonth button { margin-left: 0.5em; }

	.calendar.past .currentMonth button::after { content: '\21BA'; }
	.calendar.future .currentMonth button::after { content: '\21BB'; }

	.dayList div {
		padding: 0.3em;
	}


	.dayList {
		border-width: 0 0 0 0.05em;
	}

	.dayList .day {
		text-align: center;
	}

	.month {
		flex-direction: column;
		border-width: 0 0 0.05em 0.05em;
		overflow: hidden;
	}

	.week {
		position: relative;
		width: 100%;
		padding: 0;
		margin: 0;
		border: none;
	}

	.day {
		position: relative;
		border-style: solid;
		border-color: #DDD;
		border-width: 0.05em 0.05em 0 0;
		width: 14.285714%;
		background-color: #fff;
	}

	.day .content.draghover {
		border: 3px solid yellow;
	}

	/* Use z-index to ensure events too tall for the view are clipped vertically */

	.month .week1 { z-index: 2; }
	.month .week2 { z-index: 4; }
	.month .week3 { z-index: 6; }
	.month .week4 { z-index: 8; }
	.month .week5 { z-index: 10; }
	.month .week6 { z-index: 12; }

	.month .week1 .event { z-index: 3; }
	.month .week2 .event { z-index: 5; }
	.month .week3 .event { z-index: 7; }
	.month .week4 .event { z-index: 9; }
	.month .week5 .event { z-index: 11; }
	.month .week6 .event { z-index: 13; }

	.day.today {
		background-color: #ffe;
	}

	.day.past {
		background-color: #eee
	}

	.day.outsideOfMonth {
		background-color: #ccc;
	}

	.day .content {
		position: absolute;
		left: 0;
		top: 0;
		bottom: 0;
		right: 0;
	}

	.date {
		float: right;
		padding: 0.2em;
		clear: both;
		line-height: 1em;
	}

	.event {
		position: absolute;
		border: 1px solid #e0e0f0;
		border-radius: 0.5em;
		background-color: #e7e7ff;
		padding: 0.3em 0.3em;
		line-height: 1em;
		text-overflow: ellipsis;
		white-space: nowrap;
		overflow: hidden;
	}

	.event.purple { background-color: #f0e0ff; border-color: #e7d7f7; }
	.event.orange { background-color: #ffe7d0; border-color: #f7e0c7; }

	.event.slot1 { top: 1.5em; }
	.event.slot2 { top: calc(1.5em + 1 * 1.6em + 2px); }
	.event.slot3 { top: calc(1.5em + 2 * 1.6em + 2px); }
	.event.slot4 { top: calc(1.5em + 3 * 1.6em + 2px); }
	.event.slot5 { top: calc(1.5em + 4 * 1.6em + 2px); }
	.event.slot6 { top: calc(1.5em + 5 * 1.6em + 2px); }
	.event.slot7 { top: calc(1.5em + 6 * 1.6em + 2px); }
	.event.slot8 { top: calc(1.5em + 7 * 1.6em + 2px); }
	.event.slot9 { top: calc(1.5em + 8 * 1.6em + 2px); }
	.event.slot10 { top: calc(1.5em + 9 * 1.6em + 2px); }
	.event.slot0 { display: none; } /* More than 10 slots not currently supported */

	.event.offset0 { left: calc(.05em); }
	.event.offset1 { left: calc(14.28571429% + .05em); }
	.event.offset2 { left: calc(14.28571429% * 2 + .05em); }
	.event.offset3 { left: calc(14.28571429% * 3 + .05em); }
	.event.offset4 { left: calc(14.28571429% * 4 + .05em); }
	.event.offset5 { left: calc(14.28571429% * 5 + .05em); }
	.event.offset6 { left: calc(14.28571429% * 6 + .05em); }

	.event.hasUrl:hover {
		text-decoration: underline;
	}

	/* Used to hold when an event has spilled over to this day */
	.event.placeholder {
		z-index: 0;
		visibility: hidden;
	}
	.event.placeholder::before {
		content: ".";
	}

	.event.span1 { width: calc(14.28571429% - .05em); }
	.event.span2 { width: calc(14.28571429% * 2 - .05em); }
	.event.span3 { width: calc(14.28571429% * 3 - .05em); text-align: center;}
	.event.span4 { width: calc(14.28571429% * 4 - .05em); text-align: center;}
	.event.span5 { width: calc(14.28571429% * 5 - .05em); text-align: center;}
	.event.span6 { width: calc(14.28571429% * 6 - .05em); text-align: center;}
	.event.span7 { width: calc(14.28571429% * 7 - .05em); text-align: center;}

	.event.continued		{ 
		border-left-style: none;
		border-top-left-radius: 0;
		border-bottom-left-radius: 0;
	}

	.event.continued::before,
	.event.toBeContinued::after { 
		content: " \21e2 ";
		color: #999;
	}

	.event.toBeContinued	{ 
		border-right-style: none; 
		border-top-right-radius: 0;
		border-bottom-right-radius: 0;
	}

	/* Set min-height to 85% of width */
	.month .day:before {
		content: "";
		display: block;
		padding-top: 85%;
	}

	/* Set min-height to 75% of width (4:3 ratio) */
	.aspect43 .month .day:before {
		padding-top: 75%;
	}

</style>
