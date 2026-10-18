<template>
	<view class="page">
		<!-- 头部（连续点击 5 次触发暗门） -->
		<view class="header">
			<text class="header-title" @click="handleSecretTap">体重记录</text>
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
					@click="handleDayClick(cell.dateStr)"
				>
					<text class="day-num">{{ cell.day }}</text>
					<view v-if="cell.hasRecord" class="day-weight-row">
						<text class="day-weight">{{ cell.weight }}</text>
						<!-- 小格子里的箭头依然保留（因为格子太小写不下文字） -->
						<text
							v-if="cell.trend"
							class="day-trend"
							:class="'trend-' + cell.trend"
						>{{ cell.trend === 'up' ? '↑' : cell.trend === 'down' ? '↓' : '→' }}</text>
					</view>
				</view>
			</view>
		</view>

		<!-- 记录详情卡片 -->
		<view class="record-card">
			<view class="card-header-row">
				<text class="selected-date">{{ selectedDateLabel }}</text>
				<text v-if="isDevMode" class="dev-badge">测试模式</text>
			</view>

			<view class="weight-display">
				<template v-if="currentWeight !== null">
					<text class="weight-value">{{ currentWeight.toFixed(1) }}</text>
					<text class="weight-unit">斤</text>

					<!-- 大卡片里改成直白的文字标签 -->
					<view v-if="trendInfo" class="trend-badge" :class="'trend-' + trendInfo.type">
						<text>{{ trendInfo.text }}</text>
					</view>
				</template>
				<text v-else class="weight-empty">暂无记录</text>
			</view>

			<!-- 提示 -->
			<view v-if="!isDevMode" class="locked-hint">
				<text v-if="isToday && currentWeight !== null">今日记录已完成，不可更改</text>
				<text v-else-if="!isToday && currentWeight !== null">历史记录，不可修改</text>
				<text v-else-if="!isToday && currentWeight === null">暂无记录，仅可查看</text>
				<text v-else>点击日历上的“今天”即可记录</text>
			</view>
			<view v-else class="locked-hint">
				<text>测试模式：点击任意日期可修改</text>
			</view>
		</view>

		<!-- 🚪 开发者测试面板 -->
		<view v-if="isDevMode" class="dev-panel">
			<view class="dev-actions">
				<view class="dev-btn" @click="mockData">注入 Mock 数据</view>
				<view class="dev-btn warning" @click="clearAllData">清空数据</view>
				<view class="dev-btn" @click="exitDevMode">退出测试</view>
			</view>
		</view>

		<view class="footer-spacer"></view>

		<!-- 💡 自定义输入弹窗 -->
		<view v-if="showModal" class="custom-modal-mask" @click="closeModal">
			<view class="custom-modal-box" @click.stop>
				<view class="modal-title">记录体重</view>
				<text class="modal-subtitle">{{ selectedDateLabel }}</text>

				<input
					class="modal-input"
					type="digit"
					v-model="modalInputValue"
					placeholder="请输入体重"
					focus
				/>

				<view class="modal-actions">
					<view class="modal-btn cancel" @click="closeModal">取消</view>
					<view class="modal-btn confirm" @click="submitModal">保存</view>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
export default {
	data() {
		return {
			records: {},
			currentYear: new Date().getFullYear(),
			currentMonth: new Date().getMonth(),
			selectedDateStr: '',
			weekdays: ['一', '二', '三', '四', '五', '六', '日'],
			storageKey: 'weight_records_v2',

			isDevMode: false,
			secretTapCount: 0,
			secretTapTimer: null,

			showModal: false,
			modalInputValue: ''
		}
	},
	computed: {
		calendarCells() {
			const cells = []
			const firstDay = new Date(this.currentYear, this.currentMonth, 1).getDay()
			const leadingEmpty = firstDay === 0 ? 6 : firstDay - 1
			const daysInMonth = new Date(this.currentYear, this.currentMonth + 1, 0).getDate()
			const todayStr = this.formatDate(new Date())
			const sortedDates = Object.keys(this.records).sort()

			for (let i = 0; i < leadingEmpty; i++) {
				cells.push({ day: '', classes: 'empty', dateStr: '', hasRecord: false, weight: '', trend: null })
			}

			for (let d = 1; d <= daysInMonth; d++) {
				const dateStr = `${this.currentYear}-${String(this.currentMonth + 1).padStart(2, '0')}-${String(d).padStart(2, '0')}`
				let classes = ''
				if (dateStr === this.selectedDateStr) classes += ' selected'
				if (dateStr === todayStr && dateStr !== this.selectedDateStr) classes += ' today'

				const record = this.records[dateStr]
				let trend = null

				if (record !== undefined) {
					const idx = sortedDates.indexOf(dateStr)
					if (idx > 0) {
						const prevWeight = this.records[sortedDates[idx - 1]].weight
						const diff = Math.round((record.weight - prevWeight) * 10) / 10
						if (diff > 0) trend = 'up'
						else if (diff < 0) trend = 'down'
						else trend = 'same'
					}
				}

				cells.push({
					day: d,
					dateStr,
					hasRecord: record !== undefined,
					weight: record !== undefined ? record.weight.toFixed(1) : '',
					trend: trend,
					classes
				})
			}
			return cells
		},
		currentWeight() {
			const r = this.records[this.selectedDateStr]
			return r !== undefined ? r.weight : null
		},
		// 计算直白的涨跌文字
		trendInfo() {
			if (this.currentWeight === null) return null

			const existingDates = Object.keys(this.records).sort()
			let lastWeight = null

			for (let i = existingDates.length - 1; i >= 0; i--) {
				if (existingDates[i] < this.selectedDateStr) {
					lastWeight = this.records[existingDates[i]].weight
					break
				}
			}

			if (lastWeight === null) {
				return { type: 'same', text: '首次记录' }
			}

			const diff = Math.round((this.currentWeight - lastWeight) * 10) / 10

			if (diff > 0) {
				return { type: 'up', text: `增重 ${diff.toFixed(1)} 斤` }
			} else if (diff < 0) {
				return { type: 'down', text: `减重 ${Math.abs(diff).toFixed(1)} 斤` }
			} else {
				return { type: 'same', text: '体重持平' }
			}
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
	},
	methods: {
		formatDate(date) {
			const y = date.getFullYear()
			const m = String(date.getMonth() + 1).padStart(2, '0')
			const d = String(date.getDate()).padStart(2, '0')
			return `${y}-${m}-${d}`
		},

		handleDayClick(dateStr) {
			if (!dateStr) return

			this.selectedDateStr = dateStr

			const isToday = dateStr === this.formatDate(new Date())
			const hasRecord = this.records[dateStr] !== undefined

			if (isToday || this.isDevMode) {
				if (!this.isDevMode && hasRecord) {
					uni.showToast({ title: '今日记录已完成，不可更改', icon: 'none' })
					return
				}

				this.modalInputValue = hasRecord ? String(this.records[dateStr].weight) : ''
				this.showModal = true
			} else {
				if (!hasRecord) {
					uni.showToast({ title: '暂无记录，仅可查看', icon: 'none' })
				}
			}
		},

		closeModal() {
			this.showModal = false
			this.modalInputValue = ''
		},

		async submitModal() {
			const val = this.modalInputValue.trim()
			if (!val) {
				uni.showToast({ title: '请输入体重', icon: 'none' })
				return
			}
			const inputWeight = parseFloat(val)
			if (isNaN(inputWeight) || inputWeight <= 0 || inputWeight > 1000) {
				uni.showToast({ title: '请输入有效数字', icon: 'none' })
				return
			}

			this.showModal = false

			const existingDates = Object.keys(this.records).sort()
			const lastWeight = existingDates.length > 0
				? this.records[existingDates[existingDates.length - 1]].weight
				: null

			const unit = this.detectUnit(inputWeight, lastWeight)
			let finalWeight
			let convertedMsg = ''

			if (unit === 'kg') {
				finalWeight = Math.round(inputWeight * 2 * 10) / 10
				convertedMsg = `已按公斤换算：${inputWeight}kg = ${finalWeight}斤`
			} else if (unit === 'jin') {
				finalWeight = Math.round(inputWeight * 10) / 10
			} else {
				const confirmed = await this.confirmUnit(inputWeight)
				if (confirmed === 'cancel') return
				if (confirmed === 'kg') {
					finalWeight = Math.round(inputWeight * 2 * 10) / 10
					convertedMsg = `已按公斤换算：${inputWeight}kg = ${finalWeight}斤`
				} else {
					finalWeight = Math.round(inputWeight * 10) / 10
				}
			}

			this.records = {
				...this.records,
				[this.selectedDateStr]: { weight: finalWeight }
			}
			uni.setStorageSync(this.storageKey, JSON.stringify(this.records))
			this.modalInputValue = ''

			if (convertedMsg) {
				uni.showToast({ title: convertedMsg, icon: 'none', duration: 2500 })
			} else {
				uni.showToast({ title: '已保存', icon: 'success' })
			}
		},

		detectUnit(inputWeight, lastWeight) {
			const MIN_JIN = 60
			const MAX_JIN = 500
			const asJin = inputWeight
			const asKg = inputWeight * 2
			const jinValid = asJin >= MIN_JIN && asJin <= MAX_JIN
			const kgValid = asKg >= MIN_JIN && asKg <= MAX_JIN

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
					success: (res) => resolve(res.confirm ? 'kg' : 'jin'),
					fail: () => resolve('cancel')
				})
			})
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

		handleSecretTap() {
			if (this.isDevMode) return
			this.secretTapCount++
			clearTimeout(this.secretTapTimer)
			this.secretTapTimer = setTimeout(() => { this.secretTapCount = 0 }, 2000)
			if (this.secretTapCount >= 5) {
				this.secretTapCount = 0
				this.isDevMode = true
				uni.vibrateShort()
				uni.showToast({ title: '已进入测试模式', icon: 'none' })
			}
		},
		exitDevMode() {
			this.isDevMode = false
			uni.showToast({ title: '已退出测试模式', icon: 'none' })
		},
		clearAllData() {
			uni.showModal({
				title: '清空数据',
				content: '确定要清空所有本地体重记录吗？',
				success: (res) => {
					if (res.confirm) {
						this.records = {}
						uni.setStorageSync(this.storageKey, JSON.stringify({}))
						uni.showToast({ title: '已清空', icon: 'none' })
					}
				}
			})
		},
		mockData() {
			const mockRecords = {}
			const today = new Date()
			for (let i = 10; i >= 0; i--) {
				const d = new Date(today)
				d.setDate(d.getDate() - i)
				const dateStr = this.formatDate(d)
				const weight = Math.round((130 + Math.random() * 5) * 10) / 10
				mockRecords[dateStr] = { weight }
			}
			this.records = mockRecords
			uni.setStorageSync(this.storageKey, JSON.stringify(this.records))
			uni.showToast({ title: '注入成功', icon: 'success' })
		}
	}
}
</script>

<style scoped>
.page { min-height: 100vh; background-color: #f5f7fa; padding-bottom: 20rpx; }
.header { display: flex; justify-content: space-between; align-items: center; padding: 24rpx 32rpx; background-color: #ffffff; }
.header-title { font-size: 44rpx; font-weight: 600; color: #1a1a1a; }
.header-sub { font-size: 26rpx; color: #007aff; font-weight: 500; background-color: #eef6ff; padding: 8rpx 24rpx; border-radius: 40rpx; }
.calendar-section { background-color: #ffffff; padding: 0 24rpx 24rpx 24rpx; margin-bottom: 24rpx; }
.month-nav { display: flex; justify-content: space-between; align-items: center; padding: 20rpx 0; }
.month-year { font-size: 36rpx; font-weight: 600; color: #1a1a1a; }
.nav-buttons { display: flex; gap: 16rpx; }
.nav-btn { width: 72rpx; height: 72rpx; border-radius: 50%; background-color: #f2f2f7; display: flex; align-items: center; justify-content: center; font-size: 36rpx; color: #007aff; font-weight: 500; }
.nav-btn:active { background-color: #e5e5ea; }
.weekdays { display: grid; grid-template-columns: repeat(7, 1fr); text-align: center; font-size: 26rpx; color: #8e8e93; padding: 12rpx 0; }
.days-grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 8rpx; padding: 8rpx 0; }
.day-cell { aspect-ratio: 1 / 1; display: flex; flex-direction: column; align-items: center; justify-content: center; border-radius: 16rpx; position: relative; border: 2rpx solid transparent; padding: 4rpx 0; box-sizing: border-box; }
.day-cell.empty { background-color: transparent; }
.day-cell.selected { background-color: #007aff; }
.day-cell.today { border-color: #007aff; }
.day-num { font-size: 32rpx; font-weight: 600; color: #1a1a1a; line-height: 1.1; }
.day-cell.selected .day-num { color: #ffffff; }
.day-cell.today .day-num { color: #007aff; }
.day-weight-row { display: flex; align-items: center; gap: 2rpx; margin-top: 2rpx; }
.day-weight { font-size: 18rpx; color: #ff9500; font-weight: 400; line-height: 1.1; letter-spacing: -0.5rpx; }
.day-cell.selected .day-weight { color: #ffe0b3; }
.day-trend { font-size: 18rpx; font-weight: 700; line-height: 1.1; }
.day-cell.selected .day-trend { color: #ffffff !important; }
.trend-up { color: #ff3b30; }
.trend-down { color: #34c759; }
.trend-same { color: #8e8e93; }

/* 记录卡片 */
.record-card { margin: 0 24rpx 24rpx; padding: 32rpx; background-color: #ffffff; border-radius: 28rpx; box-shadow: 0 4rpx 20rpx rgba(0, 0, 0, 0.03); }
.card-header-row { display: flex; justify-content: space-between; align-items: center; }
.selected-date { font-size: 28rpx; color: #8e8e93; font-weight: 500; }
.dev-badge { font-size: 22rpx; color: #ff3b30; background-color: #ffeceb; padding: 4rpx 12rpx; border-radius: 8rpx; font-weight: 600; }
.weight-display { display: flex; align-items: baseline; gap: 12rpx; margin: 20rpx 0 32rpx; }
.weight-value { font-size: 88rpx; font-weight: 700; color: #1a1a1a; line-height: 1; }
.weight-unit { font-size: 32rpx; color: #8e8e93; }
.weight-empty { font-size: 36rpx; color: #c6c6c8; font-style: italic; }

/* 文字徽标 */
.trend-badge { display: flex; align-items: center; margin-left: 16rpx; padding: 6rpx 16rpx; border-radius: 12rpx; font-weight: 600; font-size: 26rpx; line-height: 1; }
.trend-badge.trend-up { color: #ff3b30; background-color: #ffeceb; }
.trend-badge.trend-down { color: #34c759; background-color: #ebfaef; }
.trend-badge.trend-same { color: #8e8e93; background-color: #f2f2f7; }

.locked-hint { height: 80rpx; display: flex; align-items: center; justify-content: center; background-color: #f7f7f9; border-radius: 20rpx; font-size: 28rpx; color: #8e8e93; }
.footer-spacer { height: 40rpx; }

/* 🚪 开发者面板 */
.dev-panel { margin: 0 24rpx 24rpx; background-color: #1c1c1e; border-radius: 20rpx; padding: 20rpx; }
.dev-actions { display: flex; gap: 12rpx; }
.dev-btn { flex: 1; background-color: #2c2c2e; color: #007aff; padding: 20rpx 0; border-radius: 12rpx; font-size: 26rpx; font-weight: 500; text-align: center; }
.dev-btn:active { background-color: #3a3a3c; }
.dev-btn.warning { color: #ff3b30; }

/* 💡 自定义弹窗 */
.custom-modal-mask { position: fixed; top: 0; left: 0; right: 0; bottom: 0; background-color: rgba(0, 0, 0, 0.4); display: flex; align-items: center; justify-content: center; z-index: 9999; }
.custom-modal-box { width: 600rpx; background-color: #ffffff; border-radius: 24rpx; padding: 40rpx; display: flex; flex-direction: column; }
.modal-title { font-size: 36rpx; font-weight: 600; color: #1a1a1a; text-align: center; }
.modal-subtitle { font-size: 26rpx; color: #8e8e93; text-align: center; margin-top: 10rpx; margin-bottom: 30rpx; }
.modal-input { width: 100%; height: 100rpx; background-color: #f5f7fa; border-radius: 16rpx; text-align: center; font-size: 40rpx; font-weight: 600; color: #1a1a1a; margin-bottom: 40rpx; }
.modal-actions { display: flex; gap: 24rpx; }
.modal-btn { flex: 1; height: 88rpx; display: flex; align-items: center; justify-content: center; border-radius: 16rpx; font-size: 32rpx; font-weight: 600; }
.modal-btn.cancel { background-color: #f2f2f7; color: #8e8e93; }
.modal-btn.confirm { background-color: #007aff; color: #ffffff; }
</style>
