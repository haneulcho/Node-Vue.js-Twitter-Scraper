<template>
<div>
<v-layout justify-center>
	<v-flex xs12 sm12 md12 lg12>
		<v-alert v-model="alert.status" :type="alert.type" dense transition="scale-transition" v-html="alert.msg"></v-alert>
		<v-card :loading="loading" class="mx-auto ma-5" max-width="620">
			<v-toolbar flat color="red darken-1" dark>
				<v-icon>search</v-icon>
				<v-toolbar-title> 트위터 검색기</v-toolbar-title>
			</v-toolbar>
			<v-form ref="form" @submit.prevent="submit" class="pa-5">
				<v-menu
					v-model="calendar1"
					:close-on-content-click="false"
					:nudge-right="40"
					transition="scale-transition"
					offset-y
					full-width
					min-width="290px"
				>
					<template v-slot:activator="{ on }">
						<v-text-field v-model="form.startDate" label="시작일(from)" prepend-icon="event" readonly required v-on="on"></v-text-field>
					</template>
					<v-date-picker v-model="form.startDate" @input="calendar1 = false" :max="maxStartDate"></v-date-picker>
				</v-menu>
				<v-menu
					v-model="calendar2"
					:close-on-content-click="false"
					:nudge-right="40"
					transition="scale-transition"
					offset-y
					full-width
					min-width="290px"
				>
					<template v-slot:activator="{ on }">
						<v-text-field v-model="form.endDate" label="종료일(to)" prepend-icon="event" readonly required v-on="on"></v-text-field>
					</template>
					<v-date-picker v-model="form.endDate" @input="calendar2 = false" :max="maxEndDate"></v-date-picker>
				</v-menu>

				<v-text-field label="다음 해시태그 (예: #엘소드 → 해시태그 #엘소드 포함)" v-model="form.hashtags" clearable></v-text-field>
				<v-text-field label="다음 단어 모두 포함 (예: 무슨 일 → '무슨'과 '일' 모두 포함)" v-model="form.allOr" clearable></v-text-field>
				<v-text-field label="다음 문구 그대로 포함 (예: 깜짝 할인 → '깜짝 할인'이라는 문구를 그대로 포함)" v-model="form.allAnd" clearable></v-text-field>
				<v-text-field label="다음 단어 중 하나 이상 포함 (예: 고양이 개 → '고양이' 또는 '개' 또는 둘 다 포함)" v-model="form.or" clearable></v-text-field>
				<v-text-field label="다음 단어 제외 (예: 고양이 개 → '고양이'를 포함하지 않고 '개'를 포함하지 않음)" v-model="form.exclude" clearable></v-text-field>

				<v-btn :disabled="loading" @click="resetForm" depressed><v-icon>delete_forever</v-icon>비우기</v-btn>
				<v-btn :disabled="!formIsValid || loading" :loading="loading" depressed color="error" class="ma-4" type="submit"><v-icon>youtube_searched_for</v-icon>검색하기</v-btn>
			</v-form>
		</v-card>
		<div v-if="list.length && !loading">
			<ListTable :list="list"></ListTable>
		</div>
	</v-flex>
</v-layout>
</div>
</template>

<script>
import { splitDateRange, dateformat } from '@/helpers/splitDateRange.js'
import ListTable from '@/components/ListTable.vue'

export default {
	components: {
		ListTable
	},
	data: () => ({
		list: [],
		dateChunks: [],
		scrapeUrl: 'http://localhost:3000/scrape',
		// scrapeUrl = 'https://twit-search-scraper.herokuapp.com/scrape',
		maxCount: 0,
		loading: false,
		calendar1: false,
		calendar2: false,
		alert: {
			status: false,
			type: 'error',
			msg: ''
		},
		form: {
			startDate: '',
			endDate: '',
			hashtags: '',
			allOr: '',
			allAnd: '',
			or: '',
			exclude: ''
		},
		qstr: ''
	}),
	computed: {
		formIsValid () {
			return (
				(this.form.startDate && this.form.endDate && this.form.startDate < this.form.endDate) && (this.form.hashtags || this.form.allOr || this.form.allAnd || this.form.or || this.form.exclude)
			)
		},
		maxStartDate () {
			return dateformat(new Date(Date.now() - 86400000), "yyyy-mm-dd")
		},
		maxEndDate () {
			return dateformat(new Date(), "yyyy-mm-dd")
		}
	},
	methods: {
		resetForm () {
			this.list = []
			this.dateChunks = []
			this.maxCount = 0
			this.alert.status = false
			this.$refs.form.reset()
		},
		submit () {
			this.loading = true
			this.list = []
			this.alert.status = false
			this.qstr = ''
			if (this.form.allOr) { this.qstr += '%20' + this.form.allOr.replace(' ', '%20') }
			if (this.form.allAnd) { this.qstr += '%20"' + this.form.allAnd.replace(' ', '%20') + '"' }

			if (this.form.or) {
				let result = ''
				let orStr = '%20OR%20'
				let orArr = this.form.or.split(' ')

				for ( var i in orArr ) {
					if (i == 0) {
						result += '('
					}
					if (i == orArr.length - 1) {
						result += orArr[i] + ')'
					} else {
						result += orArr[i] + orStr
					}
				}

				this.qstr += '%20' + result
			}

			if (this.form.exclude) {
				let result = ''
				let excludeStr = '%20-'
				let excludeArr = this.form.exclude.split(' ')

				for ( var i in excludeArr ) {
					result += excludeStr + excludeArr[i]
				}

				this.qstr += result
			}

			if (this.form.hashtags) {
				let result = ''
				let tagStr = '%20OR%20'
				let tagArr = this.form.hashtags.replace('#', '').split(' ')

				for ( var i in tagArr ) {
					if (i == 0) {
						result += '('
					}
					if (i == tagArr.length - 1) {
						result += '%23' + tagArr[i] + ')'
					} else {
						result += '%23' + tagArr[i] + tagStr
					}
				}

				this.qstr += '%20' + result
			}

			if (this.qstr.indexOf('%20') == 0) { this.qstr = this.qstr.slice(3) }

			this.getSearchResult()
		},
		getSearchResult () {
			this.dateChunks = splitDateRange(this.form.startDate, this.form.endDate, 'day')
			this.maxCount = this.dateChunks.length
			this.sendAxios(1, 0)
		},
		sendAxios (count, idx) {
			this.setAlertMsg('info', `총 ${this.maxCount}일 중 <strong>${count}일</strong> 째 트윗을 찾는 중입니다.`)
			this.axios.post(this.scrapeUrl, { qstr: this.qstr, startDate: this.dateChunks[idx].start, endDate: this.dateChunks[idx].end })
			.then((res) => {
				this.list.push(res.data)
				this.list = this.list.reduce((acc, val) => acc.concat(val), [])
			})
			.catch(err => {
				if (err.response.status != 500) {
					this.loading = false
					this.resetForm()
					this.setAlertMsg('error', '예기치 못한 오류가 발생했습니다. 잠시 후 다시 검색해 주세요.')
				}
			})
			.then(() => {
				if (count < this.maxCount) {
					this.sendAxios(count + 1, idx + 1)
				} else if (count == this.maxCount) {
					this.loading = false
					this.setAlertMsg('success', `${this.form.startDate} ~ ${this.form.endDate} 기간 동안 총 <strong>${this.list.length}개</strong> 트윗을 찾았습니다.`)
				} else {
					this.loading = false
					this.resetForm()
					this.setAlertMsg('error', '잘못된 접근입니다.')
				}
			})
		},
		setAlertMsg (type, msg) {
			this.alert.status = true
			this.alert.type = type
			this.alert.msg = msg
			if (type == 'error') {
				setTimeout(() => {
					this.alert.status = false
				}, 3000)
			}
		}
	}
};
</script>
