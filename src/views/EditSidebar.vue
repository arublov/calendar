<!--
  - @copyright Copyright (c) 2019 Georg Ehrke <oc.list@georgehrke.com>
  - @copyright Copyright (c) 2019 Jakob Röhrl <jakob.roehrl@web.de>
  - @copyright Copyright (c) 2022 Informatyka Boguslawski sp. z o.o. sp.k., http://www.ib.pl/
  -
  - @author Georg Ehrke <oc.list@georgehrke.com>
  - @author Jakob Röhrl <jakob.roehrl@web.de>
  - @author Richard Steinmetz <richard@steinmetz.cloud>
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
	<NcAppSidebar :empty="isLoading || isError"
		:force-menu="true"
		@close="cancel">
		<template v-if="isLoading">
			<div class="app-sidebar__loading-indicator">
				<div class="icon icon-loading app-sidebar-tab-loading-indicator__icon" />
			</div>
		</template>
		<template v-else-if="isError">
			<NcEmptyContent :name="$t('calendar', 'Event does not exist')" :description="error">
				<template #icon>
					<CalendarBlank :size="20" decorative />
				</template>
			</NcEmptyContent>
		</template>

		<template v-if="!isLoading && !isError && !isNew"
			#secondary-actions>
			<NcActionLink v-if="!hideEventExport && hasDownloadURL"
				:href="downloadURL">
				<template #icon>
					<Download :size="20" decorative />
				</template>
				{{ $t('calendar', 'Export') }}
			</NcActionLink>
			<NcActionButton v-if="!canCreateRecurrenceException && !isReadOnly" @click="duplicateEvent()">
				<template #icon>
					<ContentDuplicate :size="20" decorative />
				</template>
				{{ $t('calendar', 'Duplicate') }}
			</NcActionButton>
			<NcActionButton v-if="canDelete && !canCreateRecurrenceException" @click="deleteAndLeave(false)">
				<template #icon>
					<Delete :size="20" decorative />
				</template>
				{{ $t('calendar', 'Delete') }}
			</NcActionButton>
			<NcActionButton v-if="canDelete && canCreateRecurrenceException" @click="deleteAndLeave(false)">
				<template #icon>
					<Delete :size="20" decorative />
				</template>
				{{ $t('calendar', 'Delete this occurrence') }}
			</NcActionButton>
			<NcActionButton v-if="canDelete && canCreateRecurrenceException" @click="deleteAndLeave(true)">
				<template #icon>
					<Delete :size="20" decorative />
				</template>
				{{ $t('calendar', 'Delete this and all future') }}
			</NcActionButton>
		</template>

		<template v-if="!isLoading && !isError"
			#description>
			<CalendarPickerHeader :value="selectedCalendar"
				:calendars="calendars"
				:is-read-only="isReadOnly || !canModifyCalendar"
				@update:value="changeCalendar" />

			<PropertyTitle :value="title"
				:is-read-only="isReadOnly"
				@update:value="updateTitle" />

			<PropertyTitleTimePicker :start-date="startDate"
				:start-timezone="startTimezone"
				:end-date="endDate"
				:end-timezone="endTimezone"
				:is-all-day="isAllDay"
				:is-read-only="isReadOnly"
				:can-modify-all-day="canModifyAllDay"
				:user-timezone="currentUserTimezone"
				:append-to-body="true"
				@update-start-date="updateStartDate"
				@update-start-timezone="updateStartTimezone"
				@update-end-date="updateEndDate"
				@update-end-timezone="updateEndTimezone"
				@toggle-all-day="toggleAllDay" />

			<PropertyText class="property-location"
				:is-read-only="isReadOnly"
				:prop-model="rfcProps.location"
				:value="location"
				:linkify-links="true"
				@update:value="updateLocation" />
			<PropertyText class="property-description"
				:is-read-only="isReadOnly"
				:prop-model="rfcProps.description"
				:value="description"
				:linkify-links="true"
				@update:value="updateDescription" />

			<InvitationResponseButtons v-if="isViewedByAttendee"
				:attendee="userAsAttendee"
				:calendar-id="calendarId"
				:narrow="true"
				:grow-horizontally="true"
				@close="closeEditorAndSkipAction" />
		</template>

		<NcAppSidebarTab v-if="!isLoading && !isError"
			id="app-sidebar-tab-details"
			class="app-sidebar-tab"
			:name="$t('calendar', 'Details')"
			:order="0">
			<template #icon>
				<InformationOutline :size="20" decorative />
			</template>
			<div class="app-sidebar-tab__content">
				<PropertySelect :is-read-only="isReadOnly"
					:prop-model="rfcProps.status"
					:value="status"
					@update:value="updateStatus" />
				<PropertySelect :is-read-only="isReadOnly"
					:prop-model="rfcProps.accessClass"
					:value="accessClass"
					@update:value="updateAccessClass" />
				<PropertySelect :is-read-only="isReadOnly"
					:prop-model="rfcProps.timeTransparency"
					:value="timeTransparency"
					@update:value="updateTimeTransparency" />

				<PropertySelectMultiple :colored-options="true"
					:is-read-only="isReadOnly"
					:prop-model="rfcProps.categories"
					:value="categories"
					@add-single-value="addCategory"
					@remove-single-value="removeCategory" />

				<PropertyColor :calendar-color="selectedCalendarColor"
					:is-read-only="isReadOnly"
					:prop-model="rfcProps.color"
					:value="color"
					@update:value="updateColor" />

				<AlarmList :calendar-object-instance="calendarObjectInstance"
					:is-read-only="isReadOnly" />

				<!-- TODO: If not editing the master item, force updating this and all future   -->
				<!-- TODO: You can't edit recurrence-rule of no-range recurrence-exception -->
				<Repeat :calendar-object-instance="calendarObjectInstance"
					:recurrence-rule="calendarObjectInstance.recurrenceRule"
					:is-read-only="isReadOnly"
					:is-editing-master-item="isEditingMasterItem"
					:is-recurrence-exception="isRecurrenceException"
					@force-this-and-all-future="forceModifyingFuture" />

				<AttachmentsList v-if="!isLoading"
					:calendar-object-instance="calendarObjectInstance"
					:is-read-only="isReadOnly" />

				<NcModal v-if="showModal && !isPrivate()"
					:name="t('calendar', 'Managing shared access')"
					@close="closeAttachmentsModal">
					<div class="modal-content">
						<div v-if="showPreloader" class="modal-content-preloader">
							<div :style="`width:${sharedProgress}%`" />
						</div>
						<div class="modal-h">
							{{ n('calendar', 'User requires access to your file', 'Users require access to your file', showModalUsers.length) }}
						</div>
						<div class="users">
							<NcListItemIcon v-for="attendee in showModalUsers"
								:key="attendee.uri"
								class="user-list-item"
								:name="attendee.commonName"
								:subtitle="emailWithoutMailto(attendee.uri)"
								:is-no-user="true" />
						</div>
						<div class="modal-subtitle">
							{{ n('calendar', 'Attachment requires shared access', 'Attachments requiring shared access', showModalNewAttachments.length) }}
						</div>
						<div class="attachments">
							<NcListItemIcon v-for="attachment in showModalNewAttachments"
								:key="attachment.xNcFileId"
								class="attachment-list-item"
								:name="getBaseName(attachment.fileName)"
								:url="getPreview(attachment)"
								:force-display-actions="false" />
						</div>
						<div class="modal-footer">
							<div class="modal-footer-checkbox">
								<NcCheckboxRadioSwitch v-if="!isPrivate()" :checked.sync="doNotShare">
									{{ t('calendar', 'Deny access') }}
								</NcCheckboxRadioSwitch>
							</div>
							<div class="modal-footer-buttons">
								<NcButton @click="closeAttachmentsModal">
									{{ t('calendar', 'Cancel') }}
								</NcButton>
								<NcButton type="primary"
									:disabled="showPreloader"
									@click="acceptAttachmentsModal(thisAndAllFuture)">
									{{ t('calendar', 'Invite') }}
								</NcButton>
							</div>
						</div>
					</div>
				</NcModal>
			</div>
			<SaveButtons v-if="showSaveButtons"
				class="app-sidebar-tab__buttons"
				:can-create-recurrence-exception="canCreateRecurrenceException"
				:is-new="isNew"
				:force-this-and-all-future="forceThisAndAllFuture"
				@save-this-only="prepareAccessForAttachments(false)"
				@save-this-and-all-future="prepareAccessForAttachments(true)" />
		</NcAppSidebarTab>
		<NcAppSidebarTab v-if="!isLoading && !isError"
			id="app-sidebar-tab-attendees"
			class="app-sidebar-tab"
			:name="$t('calendar', 'Attendees')"
			:order="1">
			<template #icon>
				<AccountMultiple :size="20" decorative />
			</template>
			<div class="app-sidebar-tab__content">
				<InviteesList v-if="!isLoading"
					:calendar-object-instance="calendarObjectInstance"
					:is-read-only="isReadOnly"
					:is-shared-with-me="isSharedWithMe" />
			</div>
			<SaveButtons v-if="showSaveButtons"
				class="app-sidebar-tab__buttons"
				:can-create-recurrence-exception="canCreateRecurrenceException"
				:is-new="isNew"
				:force-this-and-all-future="forceThisAndAllFuture"
				@save-this-only="prepareAccessForAttachments(false)"
				@save-this-and-all-future="prepareAccessForAttachments(true)" />
		</NcAppSidebarTab>
		<NcAppSidebarTab v-if="!isLoading && !isError && showResources"
			id="app-sidebar-tab-resources"
			class="app-sidebar-tab"
			:name="$t('calendar', 'Resources')"
			:order="3">
			<template #icon>
				<MapMarker :size="20" decorative />
			</template>
			<div class="app-sidebar-tab__content">
				<ResourceList v-if="!isLoading"
					:calendar-object-instance="calendarObjectInstance"
					:is-read-only="isReadOnly" />
			</div>
			<SaveButtons v-if="showSaveButtons"
				class="app-sidebar-tab__buttons"
				:can-create-recurrence-exception="canCreateRecurrenceException"
				:is-new="isNew"
				:force-this-and-all-future="forceThisAndAllFuture"
				@save-this-only="prepareAccessForAttachments(false)"
				@save-this-and-all-future="prepareAccessForAttachments(true)" />
		</NcAppSidebarTab>
	</NcAppSidebar>
</template>
<script>
import {
	NcAppSidebar,
	NcAppSidebarTab,
	NcActionLink,
	NcActionButton,
	NcEmptyContent,
	NcModal,
	NcListItemIcon,
	NcButton,
	NcCheckboxRadioSwitch,
} from '@nextcloud/vue'

import { mapState } from 'vuex'
import { generateUrl } from '@nextcloud/router'

import AlarmList from '../components/Editor/Alarm/AlarmList.vue'

import InviteesList from '../components/Editor/Invitees/InviteesList.vue'
import PropertySelect from '../components/Editor/Properties/PropertySelect.vue'
import PropertyText from '../components/Editor/Properties/PropertyText.vue'
import PropertyTitleTimePicker from '../components/Editor/Properties/PropertyTitleTimePicker.vue'
import PropertyTitle from '../components/Editor/Properties/PropertyTitle.vue'
import Repeat from '../components/Editor/Repeat/Repeat.vue'

import EditorMixin from '../mixins/EditorMixin.js'
import moment from '@nextcloud/moment'
import SaveButtons from '../components/Editor/SaveButtons.vue'
import PropertySelectMultiple from '../components/Editor/Properties/PropertySelectMultiple.vue'
import PropertyColor from '../components/Editor/Properties/PropertyColor.vue'
import ResourceList from '../components/Editor/Resources/ResourceList.vue'
import InvitationResponseButtons from '../components/Editor/InvitationResponseButtons.vue'
import AttachmentsList from '../components/Editor/Attachments/AttachmentsList.vue'
import CalendarPickerHeader from '../components/Editor/CalendarPickerHeader.vue'

import AccountMultiple from 'vue-material-design-icons/AccountMultiple.vue'
import CalendarBlank from 'vue-material-design-icons/CalendarBlank.vue'
import Delete from 'vue-material-design-icons/Delete.vue'
import Download from 'vue-material-design-icons/Download.vue'
import ContentDuplicate from 'vue-material-design-icons/ContentDuplicate.vue'
import InformationOutline from 'vue-material-design-icons/InformationOutline.vue'
import MapMarker from 'vue-material-design-icons/MapMarker.vue'

import { shareFile } from '../services/attachmentService.js'
import { Parameter } from '@nextcloud/calendar-js'
import getTimezoneManager from '../services/timezoneDataProviderService.js'
import logger from '../utils/logger.js'

export default {
	name: 'EditSidebar',
	components: {
		ResourceList,
		PropertyColor,
		PropertySelectMultiple,
		SaveButtons,
		AlarmList,
		NcAppSidebar,
		NcAppSidebarTab,
		NcActionLink,
		NcActionButton,
		NcEmptyContent,
		NcModal,
		NcListItemIcon,
		NcButton,
		NcCheckboxRadioSwitch,
		InviteesList,
		PropertySelect,
		PropertyText,
		PropertyTitleTimePicker,
		Repeat,
		AccountMultiple,
		CalendarBlank,
		Delete,
		Download,
		ContentDuplicate,
		InformationOutline,
		MapMarker,
		InvitationResponseButtons,
		AttachmentsList,
		CalendarPickerHeader,
		PropertyTitle,
	},
	mixins: [
		EditorMixin,
	],
	data() {
		return {
			thisAndAllFuture: false,
			doNotShare: false,
			showModal: false,
			showModalNewAttachments: [],
			showModalUsers: [],
			sharedProgress: 0,
			showPreloader: false,
		}
	},
	computed: {
		...mapState({
			locale: (state) => state.settings.momentLocale,
			hideEventExport: (state) => state.settings.hideEventExport,
			attachmentsFolder: state => state.settings.attachmentsFolder,
			showResources: state => state.settings.showResources,
		}),
		accessClass() {
			return this.calendarObjectInstance?.accessClass || null
		},
		categories() {
			return this.calendarObjectInstance?.categories || null
		},
		status() {
			return this.calendarObjectInstance?.status || null
		},
		timeTransparency() {
			return this.calendarObjectInstance?.timeTransparency || null
		},
		subTitle() {
			if (!this.calendarObjectInstance) {
				return ''
			}

			const userTimezone = getTimezoneManager().getTimezoneForId(this.currentUserTimezone)
			if (!userTimezone) {
				logger.warn(`User timezone not found: ${this.currentUserTimezone}`)
				return ''
			}

			const startDateInUserTz = this.calendarObjectInstance.eventComponent.startDate
				.getInTimezone(userTimezone)
				.jsDate
			return moment(startDateInUserTz).locale(this.locale).fromNow()
		},
		attachments() {
			return this.calendarObjectInstance?.attachments || null
		},
		currentUser() {
			return this.$store.getters.getCurrentUserPrincipal || null
		},
	},
	mounted() {
		window.addEventListener('keydown', this.keyboardCloseEditor)
		window.addEventListener('keydown', this.keyboardSaveEvent)
		window.addEventListener('keydown', this.keyboardDeleteEvent)
		window.addEventListener('keydown', this.keyboardDuplicateEvent)
	},
	beforeDestroy() {
		window.removeEventListener('keydown', this.keyboardCloseEditor)
		window.removeEventListener('keydown', this.keyboardSaveEvent)
		window.removeEventListener('keydown', this.keyboardDeleteEvent)
		window.removeEventListener('keydown', this.keyboardDuplicateEvent)
	},
	methods: {
		/**
		 * Updates the access-class of this event
		 *
		 * @param {string} accessClass The new access class
		 */
		updateAccessClass(accessClass) {
			this.$store.commit('changeAccessClass', {
				calendarObjectInstance: this.calendarObjectInstance,
				accessClass,
			})
		},
		/**
		 * Updates the status of the event
		 *
		 * @param {string} status The new status
		 */
		updateStatus(status) {
			this.$store.commit('changeStatus', {
				calendarObjectInstance: this.calendarObjectInstance,
				status,
			})
		},
		/**
		 * Updates the time-transparency of the event
		 *
		 * @param {string} timeTransparency The new time-transparency
		 */
		updateTimeTransparency(timeTransparency) {
			this.$store.commit('changeTimeTransparency', {
				calendarObjectInstance: this.calendarObjectInstance,
				timeTransparency,
			})
		},
		/**
		 * Adds a category to the event
		 *
		 * @param {string} category Category to add
		 */
		addCategory(category) {
			this.$store.commit('addCategory', {
				calendarObjectInstance: this.calendarObjectInstance,
				category,
			})
		},
		/**
		 * Removes a category from the event
		 *
		 * @param {string} category Category to remove
		 */
		removeCategory(category) {
			this.$store.commit('removeCategory', {
				calendarObjectInstance: this.calendarObjectInstance,
				category,
			})
		},
		/**
		 * Updates the color of the event
		 *
		 * @param {string} customColor The new color
		 */
		updateColor(customColor) {
			this.$store.commit('changeCustomColor', {
				calendarObjectInstance: this.calendarObjectInstance,
				customColor,
			})
		},
		/**
		 * Checks is the calendar event has attendees, but organizer or not
		 *
		 * @return {boolean}
		 */
		isPrivate() {
			return this.calendarObjectInstance.attendees.filter((attendee) => {
				if (this.currentUser.emailAddress.toLowerCase() !== (
					attendee.uri.split('mailto:').length === 2
						? attendee.uri.split('mailto:')[1].toLowerCase()
						: attendee.uri.toLowerCase()
				)) {
					return attendee
				}
				return false
			}).length === 0
		},
		getPreview(attachment) {
			if (attachment.xNcHasPreview) {
				return generateUrl(`/core/preview?fileId=${attachment.xNcFileId}&x=100&y=100&a=0`)
			}
			return attachment.formatType ? OC.MimeType.getIconUrl(attachment.formatType) : null
		},
		acceptAttachmentsModal() {
			if (!this.doNotShare) {
				const total = this.showModalNewAttachments.length
				this.showPreloader = true
				if (!this.isPrivate()) {
					this.showModalNewAttachments.map(async (attachment, i) => {
						// console.log('Add share', attachment)
						this.sharedProgress = Math.ceil(100 * (i + 1) / total)

						// add share + change attachment
						try {
							const data = await shareFile(`${this.attachmentsFolder}${attachment.fileName}`)
							attachment.shareTypes = data?.share_type?.toString()
							if (typeof attachment.attachmentProperty.getParameter('X-NC-SHARED-TYPES') === 'undefined') {
								const xNcSharedTypes = new Parameter('X-NC-SHARED-TYPES', attachment.shareTypes)
								attachment.attachmentProperty.setParameter(xNcSharedTypes)
							}
							attachment.attachmentProperty.uri = data?.url
							attachment.uri = data?.url
							// toastify success
						} catch (e) {
							// toastify err
							console.error(e)
						}
						return attachment
					})

				} else {
					// TODO it is not possible to delete shares, because share ID needed
					/* this.showModalNewAttachments.map((attachment, i) => {
						this.sharedProgress += Math.ceil(100 * (i + 1) / total)
						return attachment
					}) */
				}
			}
			setTimeout(() => {
				this.showPreloader = false
				this.sharedProgress = 0
				this.showModal = false
				this.showModalNewAttachments = []
				this.showModalUsers = []
				this.saveEvent(this.thisAndAllFuture)
			}, 500)
			// trigger save event after make each attachment access
			// 1) if !isPrivate get attachments NOT SHARED  and SharedType is empry -> API ADD SHARE
			// 2) if isPrivate get attachments SHARED  and SharedType is not empty -> API DELETE SHARE
			// 3) update calendarObject while pending access change
			// 4) after all access changes, save Event trigger
			// 5) done
		},
		closeAttachmentsModal() {
			this.showModal = false
		},
		emailWithoutMailto(mailto) {
			return mailto.split('mailto:').length === 2
				? mailto.split('mailto:')[1].toLowerCase()
				: mailto.toLowerCase()
		},
		getBaseName(name) {
			return name.split('/').pop()
		},
		prepareAccessForAttachments(thisAndAllFuture = false) {
			this.thisAndAllFuture = thisAndAllFuture
			const newAttachments = this.calendarObjectInstance.attachments.filter(attachment => {
				// get only new attachments
				// TODO get NOT only new attachments =) Maybe we should filter all attachments without share-type, 'cause event can be private and AFTER save owner could add new participant
				return !this.isPrivate() ? attachment.isNew && attachment.shareTypes === null : attachment.isNew && attachment.shareTypes !== null
			})
			// if there are new attachment and event not saved
			if (newAttachments.length > 0 && !this.isPrivate()) {
				// and is event NOT private,
				// then add share to each attachment
				// only if attachment['share-types'] is null or empty
				this.showModal = true
				this.showModalNewAttachments = newAttachments
				this.showModalUsers = this.calendarObjectInstance.attendees.filter((attendee) => {
					if (this.currentUser.emailAddress.toLowerCase() !== this.emailWithoutMailto(attendee.uri)) {
						return attendee
					}
					return false
				})
			} else {
				this.saveEvent(thisAndAllFuture)
			}
		},
		saveEvent(thisAndAllFuture = false) {
			// if there is new attachments and !private, then make modal with users and files/
			// maybe check shared access before add file
			this.saveAndLeave(thisAndAllFuture)
			this.calendarObjectInstance.attachments = this.calendarObjectInstance.attachments.map(attachment => {
				if (attachment.isNew) {
					delete attachment.isNew
				}
				return attachment
			})
		},
	},
}
</script>

<style lang="scss" scoped>
.modal-content {
	padding: 16px;
	position: relative;

	.modal-content-preloader {
		position: absolute;
		top:0;
		left:0;
		right:0;
		height: 6px;

		div {
			position: absolute;
			top:0;
			left: 0;
			background: var(--color-primary-element);
			height: 6px;
			transition: width 0.3s linear;
		}
	}
}
.modal-subtitle {
	font-weight: bold;
	font-size: 16px;
	margin-top: 16px;
}
.modal-h {
	font-size: 24px;
	font-weight: bold;
	margin: 10px 0;
}
.modal-footer {
	display: flex;
	align-items: center;
	justify-content: space-between;

	.modal-footer-buttons {
		display: flex;

		:first-child {
			margin-right: 6px;
		}
	}
}
.attachments, .users {
	display: flex;
	flex-wrap: wrap;
}
::v-deep .attachments .avatardiv img {
	border-radius: 0;
}
.attachment-list-item, .user-list-item {
	width: 50%
}
.attachment-icon {
	width: 40px;
	height: auto;
	border-radius: var(--border-radius);
}
::v-deep .app-sidebar-header__description {
	flex-direction: column;
}
:deep(.app-sidebar-header__info) {
	display: none !important;
}
.property-location {
	margin-top: 10px;
}
.property-description {
	margin-bottom: 10px;
}
</style>
