<!--
 - @copyright Copyright (c) 2020 John Molakvoæ <skjnldsv@protonmail.com>
 -
 - @author John Molakvoæ <skjnldsv@protonmail.com>
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
	<NcAppContent v-if="isLoadingForm">
		<NcEmptyContent :title="t('forms', 'Loading {title} …', { title: form.title })">
			<template #icon>
				<NcLoadingIcon :size="64" />
			</template>
		</NcEmptyContent>
	</NcAppContent>

	<NcAppContent v-else :class="{'app-content--public': publicView}">
		<TopBar v-if="!publicView"
			:permissions="form?.permissions"
			:sidebar-opened="sidebarOpened"
			@update:sidebarOpened="onSidebarChange"
			@share-form="onShareForm" />

		<!-- Forms title & description-->
		<header>
			<h2 ref="title" class="form-title">
				{{ formTitle }}
			</h2>
			<!-- eslint-disable-next-line vue/no-v-html -->
			<div v-if="!loading && !success && !!formDescription" class="form-desc" v-html="formDescription" />
			<!-- Show expiration message-->
			<p v-if="form.expires && form.showExpiration" class="info-message">
				{{ expirationMessage }}
			</p>
			<!-- Generate form information message-->
			<p v-if="infoMessage" class="info-message">
				{{ infoMessage }}
			</p>
		</header>

		<NcEmptyContent v-if="loading"
			:title="t('forms', 'Submitting form …')">
			<template #icon>
				<NcLoadingIcon :size="64" />
			</template>
		</NcEmptyContent>
		<NcEmptyContent v-else-if="success || !form.canSubmit"
			:title="t('forms', 'Thank you for completing the form!')">
			<template #icon>
				<IconCheck :size="64" />
			</template>
		</NcEmptyContent>
		<NcEmptyContent v-else-if="isExpired"
			:title="t('forms', 'Form expired')"
			:description="t('forms', 'This form has expired and is no longer taking answers')">
			<template #icon>
				<IconCheck :size="64" />
			</template>
		</NcEmptyContent>

		<!-- Questions list -->
		<form v-else
			ref="form"
			@submit.prevent="onSubmit">
			<ul>
				<Questions :is="answerTypes[question.type].component"
					v-for="(question, index) in validQuestions"
					ref="questions"
					:key="question.id"
					:read-only="true"
					:answer-type="answerTypes[question.type]"
					:index="index + 1"
					:max-string-lengths="maxStringLengths"
					v-bind="question"
					:values.sync="answers[question.id]"
					@keydown.enter="onKeydownEnter"
					@keydown.ctrl.enter="onKeydownCtrlEnter" />
			</ul>
			<input ref="submitButton"
				class="primary"
				type="submit"
				:value="t('forms', 'Submit')"
				:disabled="loading"
				:aria-label="t('forms', 'Submit form')">
		</form>
	</NcAppContent>
</template>

<script>
import { loadState } from '@nextcloud/initial-state'
import { generateOcsUrl } from '@nextcloud/router'
import { showError } from '@nextcloud/dialogs'
import { emit } from '@nextcloud/event-bus'
import axios from '@nextcloud/axios'
import moment from '@nextcloud/moment'
import NcAppContent from '@nextcloud/vue/dist/Components/NcAppContent.js'
import NcEmptyContent from '@nextcloud/vue/dist/Components/NcEmptyContent.js'
import NcLoadingIcon from '@nextcloud/vue/dist/Components/NcLoadingIcon.js'
import IconCheck from 'vue-material-design-icons/Check.vue'

import answerTypes from '../models/AnswerTypes.js'
import logger from '../utils/Logger.js'

import Question from '../components/Questions/Question.vue'
import QuestionLong from '../components/Questions/QuestionLong.vue'
import QuestionShort from '../components/Questions/QuestionShort.vue'
import QuestionMultiple from '../components/Questions/QuestionMultiple.vue'
import TopBar from '../components/TopBar.vue'
import SetWindowTitle from '../utils/SetWindowTitle.js'
import ViewsMixin from '../mixins/ViewsMixin.js'

export default {
	name: 'Submit',

	components: {
		IconCheck,
		NcAppContent,
		NcEmptyContent,
		NcLoadingIcon,
		Question,
		QuestionLong,
		QuestionShort,
		QuestionMultiple,
		TopBar,
	},

	mixins: [ViewsMixin],

	props: {
		isLoggedIn: {
			type: Boolean,
			required: false,
			default: true,
		},
		shareHash: {
			type: String,
			default: '',
		},
	},

	data() {
		return {
			maxStringLengths: loadState('forms', 'maxStringLengths'),
			answerTypes,
			answers: {},
			loading: false,
			success: false,
		}
	},

	computed: {
		validQuestions() {
			return this.form.questions.filter(question => {
				// All questions must have a valid title
				if (question.text?.trim() === '') {
					return false
				}

				// If specific conditions provided, test against them
				if ('validate' in answerTypes[question.type]) {
					return answerTypes[question.type].validate(question)
				}
				return true
			})
		},

		isRequiredUsed() {
			return this.form.questions.reduce((isUsed, question) => isUsed || question.isRequired, false)
		},

		/**
		 * Check if form is expired
		 */
		isExpired() {
			return this.form.expires && moment().unix() > this.form.expires
		},

		infoMessage() {
			let message = ''
			if (this.form.isAnonymous) {
				message += t('forms', 'Responses are anonymous.')
			}
			if (!this.form.isAnonymous && this.isLoggedIn) {
				message += t('forms', 'Responses are connected to your account.')
			}
			if (this.isRequiredUsed) {
				message += ' ' + t('forms', 'An asterisk (*) indicates mandatory questions.')
			}

			return message
		},

		expirationMessage() {
			const relativeDate = moment(this.form.expires, 'X').fromNow()
			if (this.isExpired) {
				return t('forms', 'Expired {relativeDate}.', { relativeDate })
			}
			return t('forms', 'Expires {relativeDate}.', { relativeDate })
		},
	},

	watch: {
		hash() {
			// If public view, abort. Should normally not occur.
			if (this.publicView) {
				logger.error('Hash changed on public View. Aborting.')
				return
			}
			this.resetData()
			// Fetch full form on change
			this.fetchFullForm(this.form.id)
			SetWindowTitle(this.formTitle)
		},
	},

	beforeMount() {
		// Public Views get their form by initial-state from parent. No fetch necessary.
		if (this.publicView) {
			this.isLoadingForm = false
		} else {
			this.fetchFullForm(this.form.id)
		}
		SetWindowTitle(this.formTitle)
	},

	methods: {
		/**
		 * On Enter, focus next form-element
		 * Last form element is the submit button, the form submits on enter then
		 *
		 * @param {object} event The fired event.
		 */
		onKeydownEnter(event) {
			const formInputs = Array.from(this.$refs.form)
			const sourceInputIndex = formInputs.findIndex(input => input === event.originalTarget)

			// Focus next form element
			formInputs[sourceInputIndex + 1].focus()
		},

		/**
		 * Ctrl+Enter typically fires submit on forms.
		 * Some inputs do automatically, while some need explicit handling
		 */
		onKeydownCtrlEnter() {
			// Using button-click event to not bypass validity-checks and use our specified behaviour
			this.$refs.submitButton.click()
		},

		/**
		 * Submit the form after the browser validated it 🚀
		 */
		async onSubmit() {
			this.loading = true

			try {
				await axios.post(generateOcsUrl('apps/forms/api/v2.1/submission/insert'), {
					formId: this.form.id,
					answers: this.answers,
					shareHash: this.shareHash,
				})
				this.success = true
				emit('forms:last-updated:set', this.form.id)
			} catch (error) {
				logger.error('Error while submitting the form', { error })
				showError(t('forms', 'There was an error submitting the form'))
			} finally {
				this.loading = false
			}
		},

		/**
		 * Reset View-Data
		 */
		resetData() {
			this.answers = {}
			this.loading = false
			this.success = false
		},
	},

}
</script>
<style lang="scss" scoped>
@import '../scssmixins/markdownOutput';

.app-content {
	display: flex;
	align-items: center;
	flex-direction: column;

	&--public {
		// Compensate top-padding for missing topbar
		padding-top: 50px;
	}

	header,
	form {
		width: 100%;
		max-width: 750px;
		display: flex;
		flex-direction: column;
	}

	// Title & description header
	header {
		margin-bottom: 24px;
		margin-left: 56px;

		.form-title,
		.form-desc,
		.info-message {
			width: calc(100% - 56px); // margin of header, needed if screen is < 806px (max-width + margin-left)
			font-size: 100%;
			padding: 0px 16px;
			border: none;
		}
		.form-title {
			font-size: 28px;
			font-weight: bold;
			color: var(--color-main-text);
			line-height: 34px;
			min-height: 36px;
			margin: 32px 0;
			padding-bottom: 4px;
			overflow: hidden;
			text-overflow: ellipsis;
		}
		.form-desc {
			line-height: 22px;
			padding-bottom: 20px;
			resize: none;
			min-height: 42px;
			color: var(--color-text-maxcontrast);

			@include markdown-output;
		}

		.info-message {
			padding-bottom: 20px;
			margin-top: 4px;
			resize: none;
			color: var(--color-text-maxcontrast);
		}
	}

	form {
		.question {
			// Less padding needed as submit view does not have drag handles
			padding-left: 44px;
		}

		input[type=submit] {
			align-self: flex-end;
			margin: 5px;
			margin-bottom: 160px;
			padding: 10px 20px;
		}
	}
}
</style>
