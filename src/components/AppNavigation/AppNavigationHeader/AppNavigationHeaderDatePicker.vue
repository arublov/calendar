<!--
  - @copyright Copyright (c) 2019 Georg Ehrke <oc.list@georgehrke.com>
  -
  - @author Georg Ehrke <oc.list@georgehrke.com>
  -
  - @license AGPL-3.0-or-later
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<div class="datepicker-button-section">
		<NcButton v-shortkey="previousShortKeyConf"
			:aria-label="previousLabel"
			class="datepicker-button-section__previous button"
			:name="previousLabel"
			@click="navigateToPreviousTimeRange"
			@shortkey="navigateToPreviousTimeRange">
			<template #icon>
				<ChevronLeftIcon :size="22" />
			</template>
		</NcButton>
		<NcButton class="datepicker-button-section__datepicker-label button datepicker-label"
			@click.stop.prevent="toggleDatepicker"
			@mousedown.stop.prevent="doNothing"
			@mouseup.stop.prevent="doNothing">
			{{ selectedDate | formatDateRange(view, locale) }}
		</NcButton>
		<DatePicker ref="datepicker"
			class="datepicker-button-section__datepicker"
			:date="selectedDate"
			:is-all-day="true"
			:open.sync="isDatepickerOpen"
			:type="view === 'multiMonthYear' ? 'year' : 'date'"
			@change="navigateToDate" />
		<NcButton v-shortkey="nextShortKeyConf"
			:aria-label="nextLabel"
			class="datepicker-button-section__next button"
			:name="nextLabel"
			@click="navigateToNextTimeRange"
			@shortkey="navigateToNextTimeRange">
			<template #icon>
				<ChevronRightIcon :size="22" />
			</template>
		</NcButton>
	</div>
</template>

<script>
import {
	getYYYYMMDDFromDate,
	getDateFromFirstdayParam,
	modifyDate,
} from '../../../utils/date.js'
import { mapState } from 'vuex'
import formatDateRange from '../../../filters/dateRangeFormat.js'
import DatePicker from '../../Shared/DatePicker.vue'
import ChevronLeftIcon from 'vue-material-design-icons/ChevronLeft.vue'
import ChevronRightIcon from 'vue-material-design-icons/ChevronRight.vue'
import { NcButton } from '@nextcloud/vue'

export default {
	name: 'AppNavigationHeaderDatePicker',
	components: {
		DatePicker,
		ChevronLeftIcon,
		ChevronRightIcon,
		NcButton,
	},
	filters: {
		formatDateRange,
	},
	data() {
		return {
			isDatepickerOpen: false,
		}
	},
	computed: {
		...mapState({
			locale: (state) => state.settings.momentLocale,
		}),
		selectedDate() {
			return getDateFromFirstdayParam(this.$route.params?.firstDay ?? 'now')
		},
		previousShortKeyConf() {
			return {
				previous_p: ['p'],
				previous_k: ['k'],
			}
		},
		previousLabel() {
			switch (this.view) {
			case 'timeGridDay':
				return this.$t('calendar', 'Previous day')

			case 'timeGridWeek':
				return this.$t('calendar', 'Previous week')

			case 'multiMonthYear':
				return this.$t('calendar', 'Previous year')

			case 'dayGridMonth':
			default:
				return this.$t('calendar', 'Previous month')
			}
		},
		nextShortKeyConf() {
			return {
				next_j: ['j'],
				next_n: ['n'],
			}
		},
		nextLabel() {
			switch (this.view) {
			case 'timeGridDay':
				return this.$t('calendar', 'Next day')

			case 'timeGridWeek':
				return this.$t('calendar', 'Next week')

			case 'multiMonthYear':
				return this.$t('calendar', 'Next year')

			case 'dayGridMonth':
			default:
				return this.$t('calendar', 'Next month')
			}
		},
		view() {
			return this.$route.params.view
		},
	},
	methods: {
		navigateToPreviousTimeRange() {
			return this.navigateTimeRangeByFactor(-1)
		},
		navigateToNextTimeRange() {
			return this.navigateTimeRangeByFactor(1)
		},
		navigateTimeRangeByFactor(factor) {
			let newDate

			switch (this.$route.params.view) {
			case 'timeGridDay':
				newDate = modifyDate(this.selectedDate, {
					day: factor,
				})
				break

			case 'timeGridWeek':
				newDate = modifyDate(this.selectedDate, {
					week: factor,
				})
				break

			case 'multiMonthYear':
				newDate = modifyDate(this.selectedDate, {
					year: factor,
				})
				break

			case 'dayGridMonth':
			case 'listMonth':
			default: {
				// modifyDate is just adding one month, so we have to manually
				// set the date of month to 1. Otherwise if your date is set to
				// January 30th and you add one month, February 30th doesn't exist
				// and it automatically changes to March 1st. Same happens on March 31st.
				const firstDayOfCurrentMonth = new Date(this.selectedDate.getTime())
				firstDayOfCurrentMonth.setDate(1)
				newDate = modifyDate(firstDayOfCurrentMonth, {
					month: factor,
				})
				break
			}
			}

			this.navigateToDate(newDate)
		},
		navigateToDate(date) {
			const name = this.$route.name
			const params = Object.assign({}, this.$route.params, {
				firstDay: getYYYYMMDDFromDate(date),
			})

			// Don't push new route when day didn't change
			if (this.$route.params.firstDay === getYYYYMMDDFromDate(date)) {
				return
			}

			this.$router.push({ name, params })
		},
		toggleDatepicker() {
			this.isDatepickerOpen = !this.isDatepickerOpen
		},
		doNothing() {
			// This function does nothing in itself,
			// it only captures and prevents the mousedown and mouseup of vue2-datepicker
		},
	},
}
</script>
