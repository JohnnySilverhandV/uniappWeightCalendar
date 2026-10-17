<template>
	<view class="page">
		<!-- 头部 -->
		<view class="header">
			<text class="header-title">瑞哥体重记录</text>
			<text class="header-sub">{{ totalCount }} 条记录</text>
		</view>

		<!-- 日历 -->
		<view class="calendar-section">
			<view class="month-nav">
				<text class="month-year">{{ currentYear }}年{{ currentMonth + 1 }}月</text>
				<view class="nav-buttons">
					<view class="nav-btn" @click="prevMonth">←</view>
					<view class="nav-btn" @click="nextMonth">→</view>
				</view>
			</view>

			<view class="weekdays">
				<text v-for="(w, i) in weekdays" :key="i">{{ w }}</text>
			</view>

			<view class="days-grid">
				<view
					v-for="(cell, idx) in calendarCells"
					:key="idx"
					:class="['day-cell', cell.classes]"
					@click="cell.day && selectDate(cell.dateStr)"
				>
					<text class="day-num">{{ cell.day }}</text>
					<view v-if="cell.hasRecord" class="day-weight-row">
						<text class="day-weight">{{ cell.weight }}</text>
						<text
							v-if="cell.trend"
							class="day-trend"
							:class="'trend-' + cell.trend"
						>{{ cell.trend === 'up' ? '↑' : cell.trend === 'down' ? '↓' : '→' }}</text>
					</view>
				</view>
			</view>
		</view>

		<!-- 记录卡片 -->
		<view class="record-card">
			<text class="selected-date">{{ selectedDateLabel }}</text>

			<view class="weight-display">
				<template v-if="currentWeight !== null">
					<text class="weight-value">{{ currentWeight.toFixed(1) }}</text>
					<text class="weight-unit">斤</text>
					<text
						v-if="currentTrend"
						class="card-trend"
						:class="'trend-' + currentTrend"
					>{{ currentTrend === 'up' ? '↑' : currentTrend === 'down' ? '↓' : '→' }}</text>
				</template>
				<text v-else class="weight-empty">暂无记录</text>
			</view>

			<!-- 只有「今天 + 今天还没记录」才显示输入框 -->
			<view v-if="canEdit" class="input-area">
				<input
					class="weight-input"
					type="digit"
					v-model="inputValue"
					placeholder="输入体重（斤/公斤均可）"
					placeholder-class="input-placeholder"
				/>
				<view class="save-btn" :class="{ disabled: !inputValue }" @click="saveWeight">保存</view>
			</view>
			<view v-else class="locked-hint">
				<text v-if="isToday && currentWeight !== null">今日记录已完成，不可更改</text>
				<text v-else-if="!isToday && currentWeight !== null">历史记录，不可修改</text>
				<text v-else-if="!isToday && currentWeight === null">暂无记录，仅可查看</text>
				<text v-else>请选择今天来记录体重</text>
			</view>
			<text v-if="canEdit" class="hint-text">自动识别斤/公斤，存为斤</text>
		</view>

		<view class="footer-spacer"></view>
	</view>
</template>

<script>
export default {
	data() {
		return {
			// 格式：{ 'YYYY-MM-DD': { weight: 斤数, trend: 'up'|'down'|'same'|null } }
			records: {},
			currentYear: new Date().getFullYear(),
			currentMonth: new Date().getMonth(),
			selectedDateStr: '',
			inputValue: '',
			weekdays: ['一', '二', '三', '四', '五', '六', '日'],
			storageKey: 'weight_records_v2'
		}
	},
	computed: {
		calendarCells() {
			const cells = []
			const firstDay = new Date(this.currentYear, this.currentMonth, 1).getDay()
			const leadingEmpty = firstDay === 0 ? 6 : firstDay - 1
			const daysInMonth = new Date(this.currentYear, this.currentMonth + 1, 0).getDate()
			const todayStr = this.formatDate(new Date())

			for (let i = 0; i < leadingEmpty; i++) {
				cells.push({ day: '', classes: 'empty', dateStr: '', hasRecord: false, weight: '', trend: null })
			}
			for (let d = 1; d <= daysInMonth; d++) {
				const dateStr = `${this.currentYear}-${String(this.currentMonth + 1).padStart(2, '0')}-${String(d).padStart(2, '0')}`
				let classes = ''
				if (dateStr === this.selectedDateStr) classes += ' selected'
				if (dateStr === todayStr && dateStr !== this.selectedDateStr) classes += ' today'

				const record = this.records[dateStr]
				cells.push({
					day: d,
					dateStr,
					hasRecord: record !== undefined,
					weight: record !== undefined ? record.weight.toFixed(1) : '',
					trend: record !== undefined ? record.trend : null,
					classes
				})
			}
			return cells
		},
		currentWeight() {
			const r = this.records[this.selectedDateStr]
			return r !== undefined ? r.weight : null
		},
		currentTrend() {
			const r = this.records[this.selectedDateStr]
			return r !== undefined ? r.trend : null
		},
		selectedDateLabel() {
			if (!this.selectedDateStr) return ''
			const [y, m, d] = this.selectedDateStr.split('-').map(Number)
			const date = new Date(y, m - 1, d)
			const weekdays = ['星期日', '星期一', '星期二', '星期三', '星期四', '星期五', '星期六']
			return `${y}年${m}月${d}日 ${weekdays[date.getDay()]}`
		},
		totalCount() {
			return Object.keys(this.records).length
		},
		isToday() {
			return this.selectedDateStr === this.formatDate(new Date())
		},
		canEdit() {
			return this.isToday && this.currentWeight === null
		}
	},
	onLoad() {
		try {
			const stored = uni.getStorageSync(this.storageKey)
			if (stored) this.records = JSON.parse(stored)
		} catch (e) {
			console.error('读取存储失败', e)
		}
		this.selectedDateStr = this.formatDate(new Date())
		const r = this.records[this.selectedDateStr]
		if (r !== undefined) this.inputValue = String(r.weight)
	},
	methods: {
		formatDate(date) {
			const y = date.getFullYear()
			const m = String(date.getMonth() + 1).padStart(2, '0')
			const d = String(date.getDate()).padStart(2, '0')
			return `${y}-${m}-${d}`
		},
		selectDate(dateStr) {
			this.selectedDateStr = dateStr
			const r = this.records[dateStr]
			this.inputValue = r !== undefined ? String(r.weight) : ''
		},
		prevMonth() {
			this.currentMonth--
			if (this.currentMonth < 0) {
				this.currentMonth = 11
				this.currentYear--
			}
		},
		nextMonth() {
			this.currentMonth++
			if (this.currentMonth > 11) {
				this.currentMonth = 0
				this.currentYear++
			}
		},

		// ========== 单位识别 ==========
		// 返回 'jin' | 'kg' | 'unknown'
		detectUnit(inputWeight, lastWeight) {
			const MIN_JIN = 60
			const MAX_JIN = 500

			const asJin = inputWeight
			const asKg = inputWeight * 2

			const jinValid = asJin >= MIN_JIN && asJin <= MAX_JIN
			const kgValid = asKg >= MIN_JIN && asKg <= MAX_JIN

			// 有上次记录：优先按「哪个更接近上次」判断
			if (lastWeight !== null) {
				const diffJin = Math.abs(asJin - lastWeight)
				const diffKg = Math.abs(asKg - lastWeight)

				if (Math.abs(diffJin - diffKg) > 10) {
					return diffJin < diffKg ? 'jin' : 'kg'
				}
				if (jinValid && !kgValid) return 'jin'
				if (kgValid && !jinValid) return 'kg'
				return 'unknown'
			}

			// 无上次记录：只看合理范围
			if (jinValid && !kgValid) return 'jin'
			if (kgValid && !jinValid) return 'kg'
			return 'unknown'
		},

		confirmUnit(inputWeight) {
			return new Promise((resolve) => {
				uni.showModal({
					title: '确认单位',
					content: `你输入的是 ${inputWeight}，是斤还是公斤？`,
					confirmText: '公斤',
					cancelText: '斤',
					success: (res) => {
						resolve(res.confirm ? 'kg' : 'jin')
					},
					fail: () => resolve('cancel')
				})
			})
		},

		// ========== 保存 ==========
		async saveWeight() {
			if (!this.isToday) {
				uni.showToast({ title: '只能记录今天的体重', icon: 'none' })
				return
			}
			if (this.records[this.selectedDateStr] !== undefined) {
				uni.showToast({ title: '今日已记录，不可更改', icon: 'none' })
				return
			}
			const val = this.inputValue.trim()
			if (!val) {
				uni.showToast({ title: '请输入体重', icon: 'none' })
				return
			}
			const inputWeight = parseFloat(val)
			if (isNaN(inputWeight) || inputWeight <= 0 || inputWeight > 1000) {
				uni.showToast({ title: '请输入有效数字', icon: 'none' })
				return
			}

			// 拿上次体重
			const existingDates = Object.keys(this.records).sort()
			const lastWeight = existingDates.length > 0
				? this.records[existingDates[existingDates.length - 1]].weight
				: null

			// 自动识别单位
			const unit = this.detectUnit(inputWeight, lastWeight)
			let finalWeight
			let convertedMsg = ''

			if (unit === 'kg') {
				finalWeight = Math.round(inputWeight * 2 * 10) / 10
				convertedMsg = `已按公斤换算：${inputWeight}kg = ${finalWeight}斤`
			} else if (unit === 'jin') {
				finalWeight = Math.round(inputWeight * 10) / 10
			} else {
				// 模糊，弹窗确认
				const confirmed = await this.confirmUnit(inputWeight)
				if (confirmed === 'cancel') return
				if (confirmed === 'kg') {
					finalWeight = Math.round(inputWeight * 2 * 10) / 10
					convertedMsg = `已按公斤换算：${inputWeight}kg = ${finalWeight}斤`
				} else {
					finalWeight = Math.round(inputWeight * 10) / 10
				}
			}

			// 算趋势
			let trend = null
			if (lastWeight !== null) {
				const diff = Math.round((finalWeight - lastWeight) * 10) / 10
				if (diff > 0) trend = 'up'
				else if (diff < 0) trend = 'down'
				else trend = 'same'
			}

			// 存
			this.records = {
				...this.records,
				[this.selectedDateStr]: {
					weight: finalWeight,
					trend: trend
				}
			}
			uni.setStorageSync(this.storageKey, JSON.stringify(this.records))

			// 提示
			if (convertedMsg) {
				uni.showToast({ title: convertedMsg, icon: 'none', duration: 2500 })
			} else {
				uni.showToast({ title: '已保存', icon: 'success' })
			}
		}
	}
}
</script>

<style scoped>
.page {
	min-height: 100vh;
	background-color: #f5f7fa;
	padding-bottom: 20rpx;
}

/* 头部 */
.header {
	display: flex;
	justify-content: space-between;
	align-items: center;
	padding: 24rpx 32rpx;
	background-color: #ffffff;
}
.header-title {
	font-size: 44rpx;
	font-weight: 600;
	color: #1a1a1a;
}
.header-sub {
	font-size: 26rpx;
	color: #007aff;
	font-weight: 500;
	background-color: #eef6ff;
	padding: 8rpx 24rpx;
	border-radius: 40rpx;
}

/* 日历 */
.calendar-section {
	background-color: #ffffff;
	padding: 0 24rpx 24rpx 24rpx;
	margin-bottom: 24rpx;
}
.month-nav {
	display: flex;
	justify-content: space-between;
	align-items: center;
	padding: 20rpx 0;
}
.month-year {
	font-size: 36rpx;
	font-weight: 600;
	color: #1a1a1a;
}
.nav-buttons {
	display: flex;
	gap: 16rpx;
}
.nav-btn {
	width: 72rpx;
	height: 72rpx;
	border-radius: 50%;
	background-color: #f2f2f7;
	display: flex;
	align-items: center;
	justify-content: center;
	font-size: 36rpx;
	color: #007aff;
	font-weight: 500;
}
.nav-btn:active {
	background-color: #e5e5ea;
}
.weekdays {
	display: grid;
	grid-template-columns: repeat(7, 1fr);
	text-align: center;
	font-size: 26rpx;
	color: #8e8e93;
	padding: 12rpx 0;
}
.days-grid {
	display: grid;
	grid-template-columns: repeat(7, 1fr);
	gap: 8rpx;
	padding: 8rpx 0;
}
.day-cell {
	aspect-ratio: 1 / 1;
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
	border-radius: 16rpx;
	position: relative;
	border: 2rpx solid transparent;
	padding: 4rpx 0;
	box-sizing: border-box;
}
.day-cell.empty {
	background-color: transparent;
}
.day-cell.selected {
	background-color: #007aff;
}
.day-cell.today {
	border-color: #007aff;
}

/* 日期数字 */
.day-num {
	font-size: 32rpx;
	font-weight: 600;
	color: #1a1a1a;
	line-height: 1.1;
}
.day-cell.selected .day-num {
	color: #ffffff;
}
.day-cell.today .day-num {
	color: #007aff;
}

/* 体重 + 箭头 */
.day-weight-row {
	display: flex;
	align-items: center;
	gap: 2rpx;
	margin-top: 2rpx;
}
.day-weight {
	font-size: 18rpx;
	color: #ff9500;
	font-weight: 400;
	line-height: 1.1;
	letter-spacing: -0.5rpx;
}
.day-cell.selected .day-weight {
	color: #ffe0b3;
}
.day-trend {
	font-size: 18rpx;
	font-weight: 700;
	line-height: 1.1;
}
.day-cell.selected .day-trend {
	color: #ffffff !important;
}

/* 趋势颜色 */
.trend-up { color: #ff3b30; }
.trend-down { color: #34c759; }
.trend-same { color: #8e8e93; }

/* 记录卡片 */
.record-card {
	margin: 0 24rpx 24rpx;
	padding: 32rpx;
	background-color: #ffffff;
	border-radius: 28rpx;
	box-shadow: 0 4rpx 20rpx rgba(0, 0, 0, 0.03);
}
.selected-date {
	font-size: 28rpx;
	color: #8e8e93;
	font-weight: 500;
}
.weight-display {
	display: flex;
	align-items: baseline;
	gap: 12rpx;
	margin: 20rpx 0 32rpx;
}
.weight-value {
	font-size: 88rpx;
	font-weight: 700;
	color: #1a1a1a;
	line-height: 1;
}
.weight-unit {
	font-size: 32rpx;
	color: #8e8e93;
}
.card-trend {
	font-size: 40rpx;
	font-weight: 700;
	margin-left: 8rpx;
}
.weight-empty {
	font-size: 36rpx;
	color: #c6c6c8;
	font-style: italic;
}

/* 输入区 */
.input-area {
	display: flex;
	align-items: center;
	gap: 20rpx;
}
.weight-input {
	flex: 1;
	height: 96rpx;
	border: 3rpx solid #e5e5ea;
	border-radius: 20rpx;
	padding: 0 28rpx;
	font-size: 36rpx;
	font-weight: 500;
	color: #1a1a1a;
	background-color: #fafafa;
}
.input-placeholder {
	color: #c6c6c8;
	font-size: 28rpx;
	font-weight: 400;
}
.save-btn {
	width: 160rpx;
	height: 96rpx;
	background-color: #007aff;
	color: #ffffff;
	border-radius: 20rpx;
	font-size: 32rpx;
	font-weight: 600;
	display: flex;
	align-items: center;
	justify-content: center;
}
.save-btn:active {
	background-color: #0056cc;
}
.save-btn.disabled {
	background-color: #c6c6c8;
}

/* 锁定提示 */
.locked-hint {
	height: 96rpx;
	display: flex;
	align-items: center;
	justify-content: center;
	background-color: #f7f7f9;
	border-radius: 20rpx;
	font-size: 28rpx;
	color: #8e8e93;
}

/* 底部小提示 */
.hint-text {
	display: block;
	font-size: 24rpx;
	color: #8e8e93;
	margin-top: 20rpx;
	text-align: right;
}

.footer-spacer {
	height: 40rpx;
}
</style>
