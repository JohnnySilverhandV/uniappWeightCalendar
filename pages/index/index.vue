<template>
	<view class="page">
		<!-- 头部 -->
		<view class="header">
			<view class="header-left">
				<text class="header-title" @click="handleSecretTap">体重记录</text>
				<text v-if="isDevMode" class="dev-badge">测试模式</text>
			</view>
			<view class="header-right">
				<text class="io-btn import" @click="importData">导入</text>
				<text class="io-btn export" @click="exportData">导出</text>
			</view>
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
			</view>

			<view class="weight-display">
				<template v-if="currentWeight !== null">
					<text class="weight-value">{{ currentWeight.toFixed(1) }}</text>
					<text class="weight-unit">斤</text>

					<view v-if="trendInfo" class="trend-badge" :class="'trend-' + trendInfo.type">
						<text>{{ trendInfo.text }}</text>
					</view>
				</template>
				<text v-else class="weight-empty">暂无记录</text>
			</view>

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
				<view class="dev-btn" @click="mockData">注入数据</view>
				<view class="dev-btn warning" @click="clearAllData">清空数据</view>
				<view class="dev-btn" @click="exitDevMode">退出测试</view>
			</view>
		</view>

		<view class="footer-spacer"></view>

		<!-- 🔢 自定义数字键盘弹窗 -->
		<view v-if="showKeypad" class="keypad-mask" @click="closeKeypad">
			<view class="keypad-card" @click.stop>
				<!-- 顶部：日期与取消 -->
				<view class="keypad-header">
					<text class="keypad-date">{{ selectedDateLabel }}</text>
					<text class="keypad-close" @click="closeKeypad">✕</text>
				</view>

				<!-- 中间：显示输入的数值 -->
				<view class="keypad-display">
					<text class="keypad-value">{{ keypadValue || '0' }}</text>
					<text class="keypad-unit">斤</text>
				</view>
				<text class="keypad-tip">输入数值，支持斤/公斤自动识别</text>

				<!-- 底部：数字键盘 -->
				<view class="keypad-grid">
					<view class="key-btn" hover-class="key-btn-hover" @click="pressKey('1')">1</view>
					<view class="key-btn" hover-class="key-btn-hover" @click="pressKey('2')">2</view>
					<view class="key-btn" hover-class="key-btn-hover" @click="pressKey('3')">3</view>

					<view class="key-btn" hover-class="key-btn-hover" @click="pressKey('4')">4</view>
					<view class="key-btn" hover-class="key-btn-hover" @click="pressKey('5')">5</view>
					<view class="key-btn" hover-class="key-btn-hover" @click="pressKey('6')">6</view>

					<view class="key-btn" hover-class="key-btn-hover" @click="pressKey('7')">7</view>
					<view class="key-btn" hover-class="key-btn-hover" @click="pressKey('8')">8</view>
					<view class="key-btn" hover-class="key-btn-hover" @click="pressKey('9')">9</view>

					<view class="key-btn" hover-class="key-btn-hover" @click="pressKey('.')">.</view>
					<view class="key-btn" hover-class="key-btn-hover" @click="pressKey('0')">0</view>
					<view class="key-btn delete-btn" hover-class="key-btn-hover" @click="deleteKey">⌫</view>
				</view>

				<!-- 保存按钮 -->
				<view class="keypad-save-btn" :class="{ disabled: !keypadValue }" @click="submitKeypad">
					保存记录
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

			// 键盘弹窗相关
			showKeypad: false,
			keypadValue: ''
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

			if (lastWeight === null) return { type: 'same', text: '首次记录' }

			const diff = Math.round((this.currentWeight - lastWeight) * 10) / 10

			if (diff > 0) return { type: 'up', text: `增重 ${diff.toFixed(1)} 斤` }
			else if (diff < 0) return { type: 'down', text: `减重 ${Math.abs(diff).toFixed(1)} 斤` }
			else return { type: 'same', text: '体重持平' }
		},
		selectedDateLabel() {
			if (!this.selectedDateStr) return ''
			const [y, m, d] = this.selectedDateStr.split('-').map(Number)
			const date = new Date(y, m - 1, d)
			const weekdays = ['星期日', '星期一', '星期二', '星期三', '星期四', '星期五', '星期六']
			return `${y}年${m}月${d}日`
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

				// 如果已经有记录，初始化带上原值；如果是新增，清空
				this.keypadValue = hasRecord ? String(this.records[dateStr].weight) : ''
				this.showKeypad = true
			} else {
				if (!hasRecord) {
					uni.showToast({ title: '暂无记录，仅可查看', icon: 'none' })
				}
			}
		},

		// ========== 键盘交互 ==========
		pressKey(key) {
			if (key === '.') {
				// 只能有一个小数点
				if (this.keypadValue.includes('.')) return
				// 如果为空，直接按小数点，补个0
				if (this.keypadValue === '') {
					this.keypadValue = '0.'
				} else {
					this.keypadValue += '.'
				}
			} else {
				// 限制最大长度，防止输入过大
				if (this.keypadValue.length >= 6) return
				// 处理前导0（比如输入了0，再输入5，变成5而不是05）
				if (this.keypadValue === '0' && key !== '.') {
					this.keypadValue = key
				} else {
					this.keypadValue += key
				}
			}
		},

		deleteKey() {
			if (this.keypadValue.length > 0) {
				this.keypadValue = this.keypadValue.slice(0, -1)
			}
		},

		closeKeypad() {
			this.showKeypad = false
		},

		// ========== 保存与单位识别 ==========
		async submitKeypad() {
			const val = this.keypadValue.trim()
			if (!val || parseFloat(val) <= 0) {
				uni.showToast({ title: '请输入有效体重', icon: 'none' })
				return
			}
			const inputWeight = parseFloat(val)

			this.showKeypad = false

			// 找上次体重
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

			this.records = {
				...this.records,
				[this.selectedDateStr]: { weight: finalWeight }
			}
			uni.setStorageSync(this.storageKey, JSON.stringify(this.records))
			this.keypadValue = ''

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

		// ========== 导入导出功能 ==========
		exportData() {
			const dates = Object.keys(this.records).sort()
			if (dates.length === 0) {
				uni.showToast({ title: '暂无数据可导出', icon: 'none' })
				return
			}

			let csvContent = '日期,体重(斤),较上次变化\n'
			let prevWeight = null

			dates.forEach(date => {
				const weight = this.records[date].weight
				let diffText = '首次记录'

				if (prevWeight !== null) {
					const diff = Math.round((weight - prevWeight) * 10) / 10
					if (diff > 0) diffText = `增重 ${diff.toFixed(1)}`
					else if (diff < 0) diffText = `减重 ${Math.abs(diff).toFixed(1)}`
					else diffText = '持平'
				}

				csvContent += `${date},${weight.toFixed(1)},${diffText}\n`
				prevWeight = weight
			})

			// #ifdef APP-PLUS
			uni.showLoading({ title: '生成文件中...' })
			plus.io.resolveLocalFileSystemURL('_doc', (entry) => {
				const fileName = `体重记录_${new Date().getTime()}.csv`
				entry.getFile(fileName, { create: true }, (fileEntry) => {
					fileEntry.createWriter((writer) => {
						writer.onwrite = () => {
							uni.hideLoading()
							const filePath = `file://${fileEntry.fullPath}`
							uni.shareWithSystem({
								type: 'file',
								filePath: filePath,
								success: () => uni.showToast({ title: '已导出', icon: 'success' }),
								fail: () => {
									uni.openDocument({
										filePath: filePath,
										showMenu: true,
										fail: () => uni.showToast({ title: '已保存，无法打开', icon: 'none' })
									})
								}
							})
						}
						writer.onerror = () => {
							uni.hideLoading()
							uni.showToast({ title: '写入失败', icon: 'none' })
						}
						writer.write('\uFEFF' + csvContent)
					})
				}, () => uni.hideLoading())
			}, () => uni.hideLoading())
			// #endif

			// #ifdef H5
			const blob = new Blob(['\uFEFF' + csvContent], { type: 'text/csv;charset=utf-8;' })
			const link = document.createElement('a')
			link.href = URL.createObjectURL(blob)
			link.download = `weight_${new Date().getTime()}.csv`
			link.click()
			// #endif
		},

		importData() {
			// #ifdef APP-PLUS
			plus.io.chooseFile({
				type: 'file',
				success: (res) => {
					const filePath = res.files[0]
					uni.showLoading({ title: '读取中...' })
					plus.io.resolveLocalFileSystemURL(filePath, (entry) => {
						entry.file((file) => {
							const reader = new plus.io.FileReader()
							reader.onloadend = (evt) => {
								uni.hideLoading()
								this.parseCSV(evt.target.result)
							}
							reader.onerror = () => {
								uni.hideLoading()
								uni.showToast({ title: '读取失败', icon: 'none' })
							}
							reader.readAsText(file, 'utf-8')
						})
					}, () => uni.hideLoading())
				},
				fail: () => uni.showToast({ title: '已取消', icon: 'none' })
			})
			// #endif

			// #ifdef H5
			const input = document.createElement('input')
			input.type = 'file'
			input.accept = '.csv'
			input.onchange = (e) => {
				const file = e.target.files[0]
				if (!file) return
				const reader = new FileReader()
				reader.onload = (evt) => {
					this.parseCSV(evt.target.result)
				}
				reader.readAsText(file, 'utf-8')
			}
			input.click()
			// #endif
		},

		parseCSV(text) {
			const lines = text.replace(/^\uFEFF/, '').split(/\r?\n/).filter(line => line.trim() !== '')
			if (lines.length < 2) {
				uni.showToast({ title: '格式错误', icon: 'none' })
				return
			}

			const importedRecords = {}
			let successCount = 0

			for (let i = 1; i < lines.length; i++) {
				const cols = lines[i].split(',')
				if (cols.length >= 2) {
					const dateStr = cols[0].trim().replace(/^"|"$/g, '')
					const weightStr = cols[1].trim().replace(/^"|"$/g, '')
					const weight = parseFloat(weightStr)

					if (/^\d{4}-\d{2}-\d{2}$/.test(dateStr) && !isNaN(weight) && weight > 0) {
						importedRecords[dateStr] = { weight: weight }
						successCount++
					}
				}
			}

			if (successCount > 0) {
				uni.showModal({
					title: '导入确认',
					content: `成功解析 ${successCount} 条数据。将覆盖现有记录，确定吗？`,
					success: (res) => {
						if (res.confirm) {
							this.records = importedRecords
							uni.setStorageSync(this.storageKey, JSON.stringify(this.records))
							uni.showToast({ title: '导入成功', icon: 'success' })
						}
					}
				})
			} else {
				uni.showToast({ title: '无有效数据', icon: 'none' })
			}
		},

		// 测试暗门
		prevMonth() {
			this.currentMonth--
			if (this.currentMonth < 0) { this.currentMonth = 11; this.currentYear-- }
		},
		nextMonth() {
			this.currentMonth++
			if (this.currentMonth > 11) { this.currentMonth = 0; this.currentYear++ }
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
				uni.showToast({ title: '测试模式', icon: 'none' })
			}
		},
		exitDevMode() { this.isDevMode = false },
		clearAllData() {
			uni.showModal({
				title: '清空',
				content: '确定要清空吗？',
				success: (res) => {
					if (res.confirm) {
						this.records = {}
						uni.setStorageSync(this.storageKey, JSON.stringify({}))
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
		}
	}
}
</script>

<style scoped>
/* 基础样式 */
.page { min-height: 100vh; background-color: #f5f7fa; padding-bottom: 20rpx; }
.header { display: flex; justify-content: space-between; align-items: center; padding: 24rpx 32rpx; background-color: #ffffff; }
.header-left { display: flex; align-items: center; gap: 12rpx; }
.header-title { font-size: 44rpx; font-weight: 600; color: #1a1a1a; }
.header-right { display: flex; align-items: center; gap: 16rpx; }
.io-btn { font-size: 26rpx; padding: 8rpx 20rpx; border-radius: 40rpx; font-weight: 500; }
.io-btn.import { color: #34c759; background-color: #ebfaef; }
.io-btn.export { color: #007aff; background-color: #eef6ff; }

/* 日历 */
.calendar-section { background-color: #ffffff; padding: 0 24rpx 24rpx 24rpx; margin-bottom: 24rpx; }
.month-nav { display: flex; justify-content: space-between; align-items: center; padding: 20rpx 0; }
.month-year { font-size: 36rpx; font-weight: 600; color: #1a1a1a; }
.nav-buttons { display: flex; gap: 16rpx; }
.nav-btn { width: 72rpx; height: 72rpx; border-radius: 50%; background-color: #f2f2f7; display: flex; align-items: center; justify-content: center; font-size: 36rpx; color: #007aff; font-weight: 500; }
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
.day-weight { font-size: 18rpx; color: #ff9500; font-weight: 400; line-height: 1.1; }
.day-cell.selected .day-weight { color: #ffe0b3; }
.day-trend { font-size: 18rpx; font-weight: 700; }
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
.trend-badge { display: flex; align-items: center; margin-left: 16rpx; padding: 6rpx 16rpx; border-radius: 12rpx; font-weight: 600; font-size: 26rpx; line-height: 1; }
.trend-badge.trend-up { color: #ff3b30; background-color: #ffeceb; }
.trend-badge.trend-down { color: #34c759; background-color: #ebfaef; }
.trend-badge.trend-same { color: #8e8e93; background-color: #f2f2f7; }
.locked-hint { height: 80rpx; display: flex; align-items: center; justify-content: center; background-color: #f7f7f9; border-radius: 20rpx; font-size: 28rpx; color: #8e8e93; }
.footer-spacer { height: 40rpx; }

/* 测试面板 */
.dev-panel { margin: 0 24rpx 24rpx; background-color: #1c1c1e; border-radius: 20rpx; padding: 20rpx; }
.dev-actions { display: flex; gap: 12rpx; }
.dev-btn { flex: 1; background-color: #2c2c2e; color: #007aff; padding: 20rpx 0; border-radius: 12rpx; font-size: 26rpx; font-weight: 500; text-align: center; }
.dev-btn.warning { color: #ff3b30; }

/* 🔢 自定义数字键盘 */
.keypad-mask { position: fixed; top: 0; left: 0; right: 0; bottom: 0; background-color: rgba(0, 0, 0, 0.5); display: flex; justify-content: center; align-items: center; z-index: 9999; }
.keypad-card { width: 640rpx; background-color: #ffffff; border-radius: 32rpx; padding: 40rpx 32rpx; display: flex; flex-direction: column; animation: scaleUp 0.2s cubic-bezier(0.16, 1, 0.3, 1); }
@keyframes scaleUp { from { opacity: 0; transform: scale(0.9); } to { opacity: 1; transform: scale(1); } }

.keypad-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 24rpx; }
.keypad-date { font-size: 28rpx; color: #1a1a1a; font-weight: 600; }
.keypad-close { font-size: 32rpx; color: #8e8e93; padding: 8rpx; }

.keypad-display { height: 120rpx; background-color: #f5f7fa; border-radius: 20rpx; display: flex; justify-content: center; align-items: baseline; padding: 0 32rpx; margin-bottom: 12rpx; }
.keypad-value { font-size: 72rpx; font-weight: 700; color: #1a1a1a; line-height: 1; }
.keypad-unit { font-size: 32rpx; color: #8e8e93; margin-left: 8rpx; }
.keypad-tip { font-size: 22rpx; color: #ff9500; text-align: center; margin-bottom: 32rpx; }

.keypad-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16rpx; margin-bottom: 32rpx; }
.key-btn { height: 100rpx; background-color: #f5f7fa; border-radius: 16rpx; display: flex; justify-content: center; align-items: center; font-size: 40rpx; font-weight: 600; color: #1a1a1a; }
.key-btn-hover { background-color: #e5e5ea; }
.delete-btn { font-size: 32rpx; color: #ff3b30; }

.keypad-save-btn { height: 96rpx; background-color: #007aff; color: #ffffff; border-radius: 20rpx; display: flex; justify-content: center; align-items: center; font-size: 34rpx; font-weight: 600; }
.keypad-save-btn.disabled { background-color: #c6c6c8; }
.keypad-save-btn:active { background-color: #0056cc; }
</style>
