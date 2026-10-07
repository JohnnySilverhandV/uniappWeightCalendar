<template>
	<view class="page">
		<!-- 头部标题 -->
		<view class="header">
			<text class="header-title">体重记录</text>
			<text class="header-sub">{{ totalCount }} 条记录</text>
		</view>

		<!-- 日历区域 -->
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
					<!-- 日期数字 -->
					<text class="day-num">{{ cell.day }}</text>
					<!-- 体重数字（有记录才显示） -->
					<text v-if="cell.hasRecord" class="day-weight">{{ cell.weight }}</text>
				</view>
			</view>
		</view>

		<!-- 体重记录卡片 -->
		<view class="record-card">
			<text class="selected-date">{{ selectedDateLabel }}</text>

			<view class="weight-display">
				<template v-if="currentWeight !== null">
					<text class="weight-value">{{ currentWeight.toFixed(1) }}</text>
					<text class="weight-unit">斤</text>
				</template>
				<text v-else class="weight-empty">暂无记录</text>
			</view>

			<view class="input-area">
				<input
					class="weight-input"
					type="digit"
					v-model="inputValue"
					placeholder="输入体重 (斤)"
					placeholder-class="input-placeholder"
				/>
				<view class="save-btn" :class="{ disabled: !inputValue }" @click="saveWeight">保存</view>
			</view>
			<text class="hint-text">点击日期切换，输入体重保存（单位：斤）</text>
		</view>

		<!-- 最近记录 -->
		<view class="recent-section">
			<view class="section-title">
				<text>最近记录</text>
				<text class="clear-all" @click="clearAll">清除全部</text>
			</view>
			<view class="record-list">
				<view v-if="recentRecords.length === 0" class="empty-records">
					还没有记录，快去添加吧
				</view>
				<view
					v-for="(item, idx) in recentRecords"
					:key="idx"
					class="record-item"
				>
					<text class="record-date">{{ item.displayDate }}</text>
					<text class="record-weight">{{ item.weight.toFixed(1) }} 斤</text>
				</view>
			</view>
		</view>

		<view class="footer-spacer"></view>
	</view>
</template>

<script>
export default {
	data() {
		return {
			records: {},              // { 'YYYY-MM-DD': weight(斤) }
			currentYear: new Date().getFullYear(),
			currentMonth: new Date().getMonth(),
			selectedDateStr: '',
			inputValue: '',
			weekdays: ['一', '二', '三', '四', '五', '六', '日'],
			storageKey: 'weight_records_uniapp_jin'  // 换个key，避免和之前kg数据混
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
				cells.push({ day: '', classes: 'empty', dateStr: '' })
			}
			for (let d = 1; d <= daysInMonth; d++) {
				const dateStr = `${this.currentYear}-${String(this.currentMonth + 1).padStart(2, '0')}-${String(d).padStart(2, '0')}`
				let classes = ''
				if (dateStr === this.selectedDateStr) classes += ' selected'
				if (dateStr === todayStr && dateStr !== this.selectedDateStr) classes += ' today'
				const weight = this.records[dateStr]
				cells.push({
					day: d,
					dateStr,
					hasRecord: weight !== undefined,
					weight: weight !== undefined ? weight.toFixed(1) : '',
					classes
				})
			}
			return cells
		},
		currentWeight() {
			return this.records[this.selectedDateStr] !== undefined ? this.records[this.selectedDateStr] : null
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
		recentRecords() {
			const entries = Object.entries(this.records).map(([date, weight]) => ({ date, weight }))
			entries.sort((a, b) => b.date.localeCompare(a.date))
			return entries.slice(0, 7).map(item => {
				const [y, m, d] = item.date.split('-').map(Number)
				const dateObj = new Date(y, m - 1, d)
				const weekdays = ['周日', '周一', '周二', '周三', '周四', '周五', '周六']
				return {
					displayDate: `${m}月${d}日 ${weekdays[dateObj.getDay()]}`,
					weight: item.weight
				}
			})
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
		const w = this.records[this.selectedDateStr]
		if (w !== undefined) this.inputValue = String(w)
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
			const w = this.records[dateStr]
			this.inputValue = w !== undefined ? String(w) : ''
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
		saveWeight() {
			const val = this.inputValue.trim()
			if (!val) {
				uni.showToast({ title: '请输入体重', icon: 'none' })
				return
			}
			const weight = parseFloat(val)
			// 斤的范围可以大一些，比如 40 ~ 1000 斤
			if (isNaN(weight) || weight <= 0 || weight > 1000) {
				uni.showToast({ title: '请输入有效体重', icon: 'none' })
				return
			}
			const rounded = Math.round(weight * 10) / 10
			this.records = {
				...this.records,
				[this.selectedDateStr]: rounded
			}
			uni.setStorageSync(this.storageKey, JSON.stringify(this.records))
			uni.showToast({ title: '已保存', icon: 'success' })
		},
		clearAll() {
			if (Object.keys(this.records).length === 0) {
				uni.showToast({ title: '暂无记录', icon: 'none' })
				return
			}
			uni.showModal({
				title: '确认清除',
				content: '确定要清除所有体重记录吗？此操作不可恢复。',
				success: (res) => {
					if (res.confirm) {
						this.records = {}
						uni.removeStorageSync(this.storageKey)
						this.inputValue = ''
						uni.showToast({ title: '已清除', icon: 'success' })
					}
				}
			})
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

/* 日期格子：上下两行布局 */
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

/* 日期数字：大、深色、加粗 */
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

/* 体重数字：小、橙色/主题色、不用粗体，与日期明显区分 */
.day-weight {
	font-size: 20rpx;
	color: #ff9500;
	font-weight: 400;
	line-height: 1.1;
	margin-top: 2rpx;
	letter-spacing: -0.5rpx;
}
.day-cell.selected .day-weight {
	color: #ffe0b3;   /* 选中时用浅色，跟蓝色背景区分 */
}
.day-cell.today .day-weight {
	color: #ff9500;
}

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
.weight-empty {
	font-size: 36rpx;
	color: #c6c6c8;
	font-style: italic;
}
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
.hint-text {
	display: block;
	font-size: 24rpx;
	color: #8e8e93;
	margin-top: 20rpx;
	text-align: right;
}

/* 最近记录 */
.recent-section {
	margin: 0 24rpx;
}
.section-title {
	display: flex;
	justify-content: space-between;
	align-items: center;
	font-size: 34rpx;
	font-weight: 600;
	color: #1a1a1a;
	margin-bottom: 20rpx;
}
.clear-all {
	font-size: 26rpx;
	color: #ff3b30;
	background-color: #fff0ef;
	padding: 10rpx 22rpx;
	border-radius: 40rpx;
	font-weight: 500;
}
.record-list {
	background-color: #ffffff;
	border-radius: 24rpx;
	overflow: hidden;
}
.record-item {
	display: flex;
	justify-content: space-between;
	align-items: center;
	padding: 28rpx 32rpx;
	border-bottom: 2rpx solid #f0f0f0;
}
.record-item:last-child {
	border-bottom: none;
}
.record-date {
	font-size: 30rpx;
	color: #1a1a1a;
	font-weight: 500;
}
.record-weight {
	font-size: 32rpx;
	font-weight: 600;
	color: #007aff;
}
.empty-records {
	text-align: center;
	padding: 60rpx 20rpx;
	color: #8e8e93;
	font-size: 28rpx;
	background-color: #fafafa;
}
.footer-spacer {
	height: 40rpx;
}
</style>
