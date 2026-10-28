<template>
	<view class="page">
		<!-- 背景光斑：让毛玻璃有东西可虚化 -->
		<view class="bg-blobs">
			<view class="blob b1"></view>
			<view class="blob b2"></view>
			<view class="blob b3"></view>
		</view>

		<!-- 头部 -->
		<view class="header glass">
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
		<view class="calendar-section glass">
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

			<view class="days-grid" :class="gridAnimClass">
				<view
					v-for="(cell, idx) in calendarCells"
					:key="gridKey + '-' + idx"
					:class="['day-cell', cell.classes, { 'just-saved': cell.dateStr === justSavedDate }]"
					@click="handleDayClick(cell.dateStr)"
				>
					<!-- 已记录的小对勾角标 -->
					<text v-if="cell.hasRecord" class="cell-check">✓</text>
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
		<view class="record-card glass">
			<view class="card-header-row">
				<text class="selected-date">{{ selectedDateLabel }}</text>
			</view>

			<view class="weight-display">
				<template v-if="currentWeight !== null">
					<text class="weight-value">{{ displayWeight }}</text>
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

		<!-- 底部提示 -->
		<view class="foot-tip">
			<text>纯本地应用 · 不联网 · 无广告 · 无充值套路</text>
			<text>数据仅存于本机，清除缓存后记录会消失</text>
			<text>清除数据或卸载前请先「导出」备份，之后可「导入」恢复</text>
		</view>

		<!-- 🔢 自定义数字键盘弹窗（毛玻璃） -->
		<view v-if="showKeypad" class="keypad-mask" @click="closeKeypad">
			<view class="keypad-card glass" @click.stop>
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

		<!-- ✨ 保存成功：圆环逐渐变满 + 打勾（纯 CSS，兼容 App 端） -->
		<view v-if="showSuccess" class="success-mask">
			<view class="success-ring">
				<view class="ring-fill"></view>
				<view class="ring-hole"></view>
				<view class="success-check">
					<view class="check-wipe"></view>
				</view>
			</view>
			<text class="success-text">已记录</text>
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
			keypadValue: '',

			// ✨ 新增交互状态
			gridKey: 0,            // 月份切换时强制重建，触发入场动画
			gridAnimClass: '',     // 月份切换方向动画
			justSavedDate: '',     // 刚保存的日期 → 格子描边圆环动画
			showSuccess: false,    // 全屏成功动画（圆环画满 + 打勾）
			displayWeight: '',     // 数字显示
			successTimer: null
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
			return `${y}年${m}月${d}日`
		},
		isToday() {
			return this.selectedDateStr === this.formatDate(new Date())
		}
	},
	watch: {
		currentWeight: {
			immediate: true,
			handler(val) {
				this.animateWeight(val)
			}
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
	onUnload() {
		clearTimeout(this.successTimer)
		clearInterval(this._numTimer)
	},
	methods: {
		formatDate(date) {
			const y = date.getFullYear()
			const m = String(date.getMonth() + 1).padStart(2, '0')
			const d = String(date.getDate()).padStart(2, '0')
			return `${y}-${m}-${d}`
		},

		// ✨ 数字滚动动画（setInterval 实现，App/H5 双端可靠；首次显示从 0 起滚）
		animateWeight(target) {
			clearInterval(this._numTimer)
			if (target === null || target === undefined) {
				this.displayWeight = ''
				this._lastShownWeight = null
				return
			}
			const to = Math.round(target * 10) / 10
			// 从上次显示值滚动；若无上次值（首次记录/刚打开），从 0 起滚，保证动画必然可见
			const from = (this._lastShownWeight !== undefined && this._lastShownWeight !== null)
				? this._lastShownWeight : 0
			if (from === to) {
				this.displayWeight = to.toFixed(1)
				this._lastShownWeight = to
				return
			}
			const dur = 550
			const start = Date.now()
			this._numTimer = setInterval(() => {
				const p = Math.min((Date.now() - start) / dur, 1)
				const eased = 1 - Math.pow(1 - p, 3)
				this.displayWeight = (from + (to - from) * eased).toFixed(1)
				if (p >= 1) {
					clearInterval(this._numTimer)
					this._lastShownWeight = to
				}
			}, 28)
		},

		// ✨ 保存成功动画：全屏圆环画满 + 打勾 + 格子描边
		playSuccessAnimation() {
			this.justSavedDate = this.selectedDateStr
			this.showSuccess = true
			uni.vibrateShort && uni.vibrateShort()
			clearTimeout(this.successTimer)
			this.successTimer = setTimeout(() => {
				this.showSuccess = false
				this.justSavedDate = ''
			}, 1600)
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
				if (this.keypadValue.includes('.')) return
				if (this.keypadValue === '') {
					this.keypadValue = '0.'
				} else {
					this.keypadValue += '.'
				}
			} else {
				if (this.keypadValue.length >= 6) return
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

			// ✨ 播放成功动画（若带换算提示，稍作延迟让 toast 先出现）
			setTimeout(() => this.playSuccessAnimation(), convertedMsg ? 60 : 120)

			if (convertedMsg) {
				uni.showToast({ title: convertedMsg, icon: 'none', duration: 2500 })
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

		// ========== 月份切换（带方向滑动动画） ==========
		prevMonth() {
			this.gridAnimClass = 'anim-left'
			this.currentMonth--
			if (this.currentMonth < 0) { this.currentMonth = 11; this.currentYear-- }
			this.gridKey++
			setTimeout(() => { this.gridAnimClass = '' }, 200)
		},
		nextMonth() {
			this.gridAnimClass = 'anim-right'
			this.currentMonth++
			if (this.currentMonth > 11) { this.currentMonth = 0; this.currentYear++ }
			this.gridKey++
			setTimeout(() => { this.gridAnimClass = '' }, 200)
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
/* ===== 基础与背景 ===== */
.page {
	min-height: 100vh;
	background: linear-gradient(160deg, #eef2f8 0%, #e8edf5 50%, #f2ecf5 100%);
	padding-bottom: 20rpx;
}
.bg-blobs { position: fixed; top: 0; left: 0; right: 0; bottom: 0; z-index: 0; overflow: hidden; pointer-events: none; }
.blob { position: absolute; border-radius: 50%; }
.blob.b1 { width: 60vw; height: 60vw; background: #bcd4ff; top: -16vw; right: -14vw; filter: blur(70px); opacity: .6; }
.blob.b2 { width: 50vw; height: 50vw; background: #ffd9e8; bottom: -12vw; left: -12vw; filter: blur(70px); opacity: .6; }
.blob.b3 { width: 34vw; height: 34vw; background: #c8f4e0; top: 38%; left: 32%; filter: blur(70px); opacity: .55; }

/* ===== 毛玻璃通用 ===== */
.glass {
	background: rgba(255, 255, 255, 0.55);
	backdrop-filter: blur(24px) saturate(1.6);
	-webkit-backdrop-filter: blur(24px) saturate(1.6);
	border: 1rpx solid rgba(255, 255, 255, 0.65);
	box-shadow: 0 8rpx 32rpx rgba(31, 38, 135, 0.08);
}

/* ===== 头部 ===== */
.header {
	position: sticky; top: 0; z-index: 100;
	display: flex; justify-content: space-between; align-items: center;
	padding: 24rpx 32rpx; border-radius: 0 0 28rpx 28rpx;
}
.header-left { display: flex; align-items: center; gap: 12rpx; }
.header-title { font-size: 42rpx; font-weight: 700; color: #1a1a1a; letter-spacing: 1rpx; }
.header-right { display: flex; align-items: center; gap: 16rpx; }
.io-btn { font-size: 26rpx; padding: 10rpx 24rpx; border-radius: 40rpx; font-weight: 600; transition: transform .15s ease; }
.io-btn:active { transform: scale(.92); }
.io-btn.import { color: #1fad53; background-color: rgba(52, 199, 89, .16); }
.io-btn.export { color: #0071e3; background-color: rgba(0, 122, 255, .13); }
.dev-badge { font-size: 22rpx; color: #ff3b30; background-color: rgba(255,59,48,.12); padding: 4rpx 12rpx; border-radius: 8rpx; font-weight: 600; }

/* ===== 日历 ===== */
.calendar-section {
	padding: 24rpx; margin: 24rpx 24rpx 24rpx; border-radius: 32rpx;
	animation: cardIn .5s cubic-bezier(0.16, 1, 0.3, 1) both;
}
@keyframes cardIn { from { opacity: 0; transform: translateY(24rpx); } to { opacity: 1; transform: none; } }
.month-nav { display: flex; justify-content: space-between; align-items: center; padding: 4rpx 4rpx 20rpx; }
.month-year { font-size: 36rpx; font-weight: 700; color: #1a1a1a; }
.nav-buttons { display: flex; gap: 16rpx; }
.nav-btn {
	width: 76rpx; height: 76rpx; border-radius: 50%;
	background: rgba(255,255,255,.6); border: 1rpx solid rgba(255,255,255,.85);
	display: flex; align-items: center; justify-content: center;
	font-size: 36rpx; color: #0071e3; font-weight: 600;
	transition: transform .18s cubic-bezier(.34,1.56,.64,1), background .18s;
}
.nav-btn:active { transform: scale(.85); background: rgba(0,122,255,.12); }
.weekdays { display: grid; grid-template-columns: repeat(7, 1fr); text-align: center; font-size: 26rpx; color: #8e8e93; padding: 12rpx 0; }
.days-grid { display: grid; grid-template-columns: repeat(7, 1fr); gap: 8rpx; transition: opacity .2s ease, transform .2s ease; }
.days-grid.anim-left { opacity: 0; transform: translateX(-24rpx); }
.days-grid.anim-right { opacity: 0; transform: translateX(24rpx); }

.day-cell {
	aspect-ratio: 1 / 1; display: flex; flex-direction: column; align-items: center; justify-content: center;
	border-radius: 18rpx; position: relative; border: 2rpx solid transparent; padding: 4rpx 0; box-sizing: border-box;
	transition: transform .18s cubic-bezier(.34,1.56,.64,1), background .2s, box-shadow .2s;
}
.day-cell:not(.empty):active { transform: scale(.88); }
.day-cell.selected {
	background: linear-gradient(135deg, #0a84ff, #4da3ff);
	box-shadow: 0 8rpx 20rpx rgba(10,132,255,.35);
	animation: popSel .3s cubic-bezier(.34,1.56,.64,1);
}
@keyframes popSel { 0% { transform: scale(.85); } 100% { transform: scale(1); } }
.day-cell.today { border-color: #0a84ff; }
.day-cell.just-saved { animation: popSel .35s cubic-bezier(.34,1.56,.64,1); }
.day-num { font-size: 32rpx; font-weight: 600; color: #1a1a1a; line-height: 1.1; position: relative; z-index: 2; }
.day-cell.selected .day-num { color: #ffffff; }
.day-cell.today .day-num { color: #0a84ff; }
.day-weight-row { display: flex; align-items: center; gap: 2rpx; margin-top: 2rpx; position: relative; z-index: 2; }
.day-weight { font-size: 18rpx; color: #ff9500; font-weight: 400; line-height: 1.1; }
.day-cell.selected .day-weight { color: #ffe0b3; }
.day-trend { font-size: 18rpx; font-weight: 700; }
.day-cell.selected .day-trend { color: #ffffff !important; }
.trend-up { color: #ff3b30; }
.trend-down { color: #34c759; }
.trend-same { color: #8e8e93; }

/* 已记录小对勾角标（绿色圆点 + 白色 ✓，兼容 App 端） */
.cell-check {
	position: absolute; top: 4rpx; right: 4rpx; width: 26rpx; height: 26rpx; z-index: 3;
	background: #30d158; border-radius: 50%;
	color: #ffffff; font-size: 18rpx; line-height: 26rpx; text-align: center; font-weight: 700;
	animation: checkPop .4s cubic-bezier(.34,1.8,.64,1) both;
}
@keyframes checkPop { from { transform: scale(0); } to { transform: scale(1); } }

/* ===== 记录卡片 ===== */
.record-card {
	margin: 0 24rpx 24rpx; padding: 36rpx; border-radius: 32rpx;
	animation: cardIn .55s .06s cubic-bezier(0.16, 1, 0.3, 1) both;
}
.card-header-row { display: flex; justify-content: space-between; align-items: center; }
.selected-date { font-size: 28rpx; color: #8e8e93; font-weight: 500; }
.weight-display { display: flex; align-items: baseline; gap: 12rpx; margin: 20rpx 0 32rpx; min-height: 90rpx; }
.weight-value { font-size: 92rpx; font-weight: 800; color: #1a1a1a; line-height: 1; letter-spacing: -1rpx; font-variant-numeric: tabular-nums; }
.weight-unit { font-size: 32rpx; color: #8e8e93; }
.weight-empty { font-size: 36rpx; color: #c6c6c8; font-style: italic; }
.trend-badge {
	display: flex; align-items: center; margin-left: 16rpx; padding: 8rpx 18rpx; border-radius: 14rpx;
	font-weight: 700; font-size: 26rpx; line-height: 1;
	animation: checkPop .35s cubic-bezier(.34,1.56,.64,1);
}
.trend-badge.trend-up { color: #ff3b30; background-color: rgba(255,59,48,.1); }
.trend-badge.trend-down { color: #34c759; background-color: rgba(52,199,89,.12); }
.trend-badge.trend-same { color: #8e8e93; background-color: rgba(142,142,147,.12); }
.locked-hint {
	height: 80rpx; display: flex; align-items: center; justify-content: center;
	background-color: rgba(142,142,147,.09); border-radius: 20rpx; font-size: 28rpx; color: #8e8e93;
}
.footer-spacer { height: 40rpx; }

/* ===== 底部提示 ===== */
.foot-tip {
	margin: 16rpx 48rpx calc(60rpx + env(safe-area-inset-bottom));
	display: flex; flex-direction: column; align-items: center;
}
.foot-tip text {
	display: block;
	font-size: 21rpx; color: rgba(142, 142, 147, 0.78);
	line-height: 1.9; text-align: center;
}
.foot-tip text:first-child {
	font-weight: 600; color: rgba(110, 110, 115, 0.9);
	letter-spacing: 1rpx;
}

/* ===== 测试面板 ===== */
.dev-panel {
	margin: 0 24rpx 24rpx; border-radius: 24rpx; padding: 20rpx;
	background: rgba(28,28,30,.92); backdrop-filter: blur(20px); -webkit-backdrop-filter: blur(20px);
}
.dev-actions { display: flex; gap: 12rpx; }
.dev-btn { flex: 1; background: rgba(255,255,255,.1); color: #0a84ff; padding: 20rpx 0; border-radius: 14rpx; font-size: 26rpx; font-weight: 600; text-align: center; }
.dev-btn.warning { color: #ff453a; }
.dev-btn:active { opacity: .7; }

/* ===== 🔢 自定义数字键盘（毛玻璃） ===== */
.keypad-mask {
	position: fixed; top: 0; left: 0; right: 0; bottom: 0;
	background: rgba(20, 24, 40, 0.32);
	backdrop-filter: blur(10px) saturate(1.2); -webkit-backdrop-filter: blur(10px) saturate(1.2);
	display: flex; justify-content: center; align-items: center; z-index: 9999;
	animation: maskIn .2s ease both;
}
@keyframes maskIn { from { opacity: 0; } to { opacity: 1; } }
.keypad-card {
	width: 640rpx; border-radius: 36rpx; padding: 40rpx 32rpx; display: flex; flex-direction: column;
	animation: springUp .32s cubic-bezier(.16, 1.2, .3, 1) both;
}
@keyframes springUp { from { opacity: 0; transform: translateY(70rpx) scale(.92); } to { opacity: 1; transform: none; } }

.keypad-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 24rpx; }
.keypad-date { font-size: 28rpx; color: #1a1a1a; font-weight: 700; }
.keypad-close { font-size: 32rpx; color: #8e8e93; padding: 8rpx; }

.keypad-display {
	height: 120rpx; background: rgba(255,255,255,.55); border: 1rpx solid rgba(255,255,255,.85);
	border-radius: 22rpx; display: flex; justify-content: center; align-items: baseline; padding: 0 32rpx; margin-bottom: 12rpx;
}
.keypad-value { font-size: 72rpx; font-weight: 800; color: #1a1a1a; line-height: 1.5; }
.keypad-unit { font-size: 32rpx; color: #8e8e93; margin-left: 8rpx; }
.keypad-tip { font-size: 22rpx; color: #ff9500; text-align: center; margin-bottom: 32rpx; }

.keypad-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16rpx; margin-bottom: 32rpx; }
.key-btn {
	height: 100rpx; background: rgba(255,255,255,.62); border: 1rpx solid rgba(255,255,255,.85);
	border-radius: 18rpx; display: flex; justify-content: center; align-items: center;
	font-size: 40rpx; font-weight: 700; color: #1a1a1a;
	transition: transform .12s ease, background .12s ease;
}
.key-btn-hover { background: rgba(0,122,255,.12); transform: scale(.92); }
.delete-btn { font-size: 32rpx; color: #ff3b30; }

.keypad-save-btn {
	height: 96rpx; border-radius: 22rpx; display: flex; justify-content: center; align-items: center;
	font-size: 34rpx; font-weight: 700; color: #ffffff;
	background: linear-gradient(135deg, #0a84ff, #3f9dff);
	box-shadow: 0 10rpx 26rpx rgba(10,132,255,.35);
	transition: transform .15s cubic-bezier(.34,1.56,.64,1);
}
.keypad-save-btn:not(.disabled):active { transform: scale(.96); }
.keypad-save-btn.disabled { background: rgba(142,142,147,.4); box-shadow: none; }

/* ===== ✨ 保存成功：圆环逐渐画满 + 打勾 ===== */
.success-mask {
	position: fixed; top: 0; left: 0; right: 0; bottom: 0; z-index: 10000; pointer-events: none;
	background: rgba(255,255,255,.25);
	backdrop-filter: blur(6px); -webkit-backdrop-filter: blur(6px);
	display: flex; flex-direction: column; justify-content: center; align-items: center;
	animation: maskIn .2s ease both;
}
.success-ring { position: relative; width: 200rpx; height: 200rpx; }
/* 渐满圆环：conic-gradient 逐帧填充（纯 CSS，App/H5 均可渲染） */
.ring-fill {
	position: absolute; top: 0; left: 0; right: 0; bottom: 0; border-radius: 50%;
	background: conic-gradient(#30d158 0deg, rgba(48, 209, 88, 0.15) 0deg);
	animation: ringFill .75s .05s linear both;
}
@keyframes ringFill {
	0%   { background: conic-gradient(#30d158 0deg,   rgba(48,209,88,.15) 0deg); }
	5%   { background: conic-gradient(#30d158 18deg,  rgba(48,209,88,.15) 18deg); }
	10%  { background: conic-gradient(#30d158 36deg,  rgba(48,209,88,.15) 36deg); }
	15%  { background: conic-gradient(#30d158 54deg,  rgba(48,209,88,.15) 54deg); }
	20%  { background: conic-gradient(#30d158 72deg,  rgba(48,209,88,.15) 72deg); }
	25%  { background: conic-gradient(#30d158 90deg,  rgba(48,209,88,.15) 90deg); }
	30%  { background: conic-gradient(#30d158 108deg, rgba(48,209,88,.15) 108deg); }
	35%  { background: conic-gradient(#30d158 126deg, rgba(48,209,88,.15) 126deg); }
	40%  { background: conic-gradient(#30d158 144deg, rgba(48,209,88,.15) 144deg); }
	45%  { background: conic-gradient(#30d158 162deg, rgba(48,209,88,.15) 162deg); }
	50%  { background: conic-gradient(#30d158 180deg, rgba(48,209,88,.15) 180deg); }
	55%  { background: conic-gradient(#30d158 198deg, rgba(48,209,88,.15) 198deg); }
	60%  { background: conic-gradient(#30d158 216deg, rgba(48,209,88,.15) 216deg); }
	65%  { background: conic-gradient(#30d158 234deg, rgba(48,209,88,.15) 234deg); }
	70%  { background: conic-gradient(#30d158 252deg, rgba(48,209,88,.15) 252deg); }
	75%  { background: conic-gradient(#30d158 270deg, rgba(48,209,88,.15) 270deg); }
	80%  { background: conic-gradient(#30d158 288deg, rgba(48,209,88,.15) 288deg); }
	85%  { background: conic-gradient(#30d158 306deg, rgba(48,209,88,.15) 306deg); }
	90%  { background: conic-gradient(#30d158 324deg, rgba(48,209,88,.15) 324deg); }
	95%  { background: conic-gradient(#30d158 342deg, rgba(48,209,88,.15) 342deg); }
	100% { background: conic-gradient(#30d158 360deg, rgba(48,209,88,.15) 360deg); }
}
/* 中心挖空成圆环 */
.ring-hole {
	position: absolute; top: 22rpx; left: 22rpx; right: 22rpx; bottom: 22rpx;
	border-radius: 50%; background: rgba(255, 255, 255, 0.93);
}
/* 打勾：内嵌 PNG（抗锯齿圆头笔画，两端渲染一致） */
.success-check {
	position: absolute; top: 0; left: 0; right: 0; bottom: 0;
}
/* 裁剪容器：宽度从 0 长到 168rpx，实现"从左往右逐渐画出" */
.check-wipe {
	position: absolute; left: 16rpx; top: 16rpx;
	width: 0; height: 168rpx; overflow: hidden;
	background-image: url("data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAeAAAAHgCAYAAAB91L6VAAAN/klEQVR42u3bR3rk1hWAUS4BSyhbznKAZTmOeqxY6qjYgrLkkZbAHdQSMHVu52yXc7bhPOUSsAT4Uq5ukVKrxWYVXj08nMF5X0/rXgB/o0gefOG/XzwAANI6+Px/XgMAEjMEABBgABBgAECAAUCAAQABBgABBgAEGAAEGAB4I8D/fg0ASMwQAECAAUCAAYDxAvwqAJCYIQCAAAOAAAMAYwX4c3EAAGkZAgDsJcD/ehUASMwQAECAAUCAAQABBgABBgAEGAAEGAAQYADIMsCvAACJGQIA7CXA/3wFAEjMEABAgAFAgAEAAQaAggL82TgAgLQMAQAEGAAEGAAQYAAoKcD/eAUASMwQAECAAWA2AX4ZAEjMEABAgAFAgAEAAQYAAQYABBgABBgAeNcA//1lACAxQwAAAQaAmQT4M3EAAGkZAgAIMAAIMAAwXoBfAgASMwQAEGAAmEuAu5cAgMQMAQAEGAAEGAAQYAAQYABAgAFAgAEAAQYAAQYA3gjw314CABI7+HQcAEBacbwIACRmCAAgwAAgwACAAAOAAAMAAgwAAgwACDAA5Bjgv74IACRmCAAgwAAgwACAAAOAAAMA2wX4BQAgMUMAAAEGgJkE+P44AIC0Du7/ywsAQGKGAAACDAACDAAIMAAIMAAgwAAgwACAAAOAAAMAAgwA+wrwn58HABIzBAAQYAAQYABAgAFAgAGAbQL8qTgAgLQMAQAEGAAEGAAYLcB/eh4ASMwQAECAAUCAAQABBoCiAvwcAJCYIQCAAAOAAAMAAgwAJQX4j88BAIkZAgAIMAAIMAAwVoDviwMASMsQAECAAUCAAQABBihWFZbhMKxP6E78exWasDAvAQZgOxfCjTDcpW4TYzOcdoAbANKK8DbxVtsMWzoKjXlO08F9f2gASKMKqzDs2DoszHdaDAEgjTp0I8T3pj4szVmAATgd337E+J7UmLcAA5A2viIswABsfi6bOr43v46uzV+AAeb6C1fdHuJ7U2cHAgwgvvtxaBcCDCC++/kqurITAQaYgxuZxNdbsAADzEabWXxvvgXbTZYB/n0DwPbaMGRqaT/5MQSAsuN7bGVHGQb4k3EAcG5NGDK3tqf8GAJA2fG9yb7yC/CzANy9iO+zw4TYWWYMAeDuLScWXwEWYIDJq0M/wQBXdifAAOLrDViADQGg+PgKsAADTNLx17dHE45vZ4cCDDDF+HYTju+x1h5zDPDvngXg9qrQhWHiGrvMjyEAlB3fYfNZ7FSAASahLSS+rV0KMID4prewTwEGEF9vvwgwwNusCopv72e/AgwwBU1B8T22tFMBBhDftFZ2OokAXweYs4jv9aEgrZ1Ow0EdB8BM1WEoSGun03FQ//Y6wBzVoQ9DITo7nRZDAMS3jPhW9irAAOIrvggwwC3VJljiiwADiO+59Ju3ebsVYADxFV8EGOBN4osAAyTWFhTfYxfsVIABxDetxk4FGEB8xRcBBjjldfEl/wD/5jpASZowFGRlp2UyBEB889XaadEBfgagBBHfZ4aCtHZaNkMASlCHXnwRYADxPa8uVPYqwADiK74IMMAtC/Fl0gH+RBwAE1OFLgyF6DefyW5nxBAA8d1/fGt7FWAA8RVfkgT4188ATMWNMBSiD7WdzpchAFPRFhTfY0s7FWAA8U2rsVMMARBf8UWAAU5pxBcBBhDfbbR2igAD4iu+7D/ATwPkZBmGgrR2yu0YApCTOvQFxXdtpwgwIL5pdaGyV945wL96GmDf6tCHoRBdqOyVOzEEYN+OQ3UkvggwQNr4dgXF9/gtfmGvCDAgvmnjW9srAgyIr/iSc4A/HgdAYm0YCtGH2k65W4YAiO92GjtFgAHxFV8EGOCUlfiCAANpNYXF99BOEWBAfNNq7ZTdBPiXTwOMpQlDQVo7ZVfieApgDHUYCtLaKbtkCMBY8e0Lim9npwgwIL7p41vZKwIMiK/4IsAAt1SbYIkvCDAgvufSb97m7RYBBsRXfBFggDeJLwgwkFhb2N/6XrBT0gX4F08BnEcbhoI0dkpKhgCIr/giwMAEvC6+sIMAfywOgDNqwlCQlZ2yL4YAzDW+rZ0iwID4ii/zC/CTAHdShz4MhWjtlBwYAjCn+HahslcEGBBf8UWAAW5ZiC8IMJBWtQlWKfHtxZc8A7x+EuCmKnRhKEQfanslR4YAiC8IMLBHN8QXBJjzvb0sw2FYn9Cd+PcqNGFhXrxFW1B8h829YK8IMKO6cM43l24TYzOktPi6rhFgRg/vegcPqyMPLPEVXxBgzvZV82qEB9faV9Oz04gvCDBnU4/8W6q9n52J70S1dooAM2Z8e28SiK/4IsCUGV8RLttSfCGTAH/050+Qtyp0YdiDxvyLUod+T9fSGNZ2ypQZQv5We37IibD45qjb/OfUbhFgRnEhk4edCIuv+IIAz8o6o4eeCE/3RxhH4gsCzPTefkXY7w/k4vgtfmGvCDBju5HpQ1CExXdf8a3tFQEmxcMz54ehCIuv+IIAF2k5gYeiCOerFV8QYM7ncCIPRxEWX9cYCLDffvaAFF/xhSkF+GdPkJ91GCaksbO9W03smnFNMXuGIMAemNPXFBbfQztFgNmXzlsLM41va6cIMN6ARVh8xRcEWIBFmFNq8YXJB/hx8rMKw8Q19jiaiO/jfQHXyE2dnTJHhpCnppAHqwiL71niW9krAkwuFgU9YEVYfMUXbhfge+MgS10YCtHY59aqwq6JbvOZ7JbZMoR8NQU9bEVYfE/qQ22vCLAh5OxIhMVXfKHUAP/0cfLVhKEwjb3ela6g3fehtlP4P0PI31qEZ6stbO8X7BQEeEoWmzcHERZf+wYBJrFlgQH2UBZfEGD8PNjDORuv2y8IMCLsIW2/21jZKQiwh7QI22tarXsVBNjDWoTtU3xBgPHQFuFT6sJ+y1184ewBvsZ0RayuDQVqZrK/iO+1vqC9daFyX8LZHNz7k2tMWxOGAjWF760OfUH76kLlfoSzMwQRFuH0FuILGIIIi3Ba1SZYpeynF18QYERYfNPHt3bfgQAjwuIrviDAiLAIn3JDfIFTAf5IHBSpCUOBmgnuoi1sB0v3F2zPEERYhMW39P8AgQAjwjMLgfgCAizCImzm4gsCjCCUHYbSZt26f2CMAP/4GvPRhKFAjRmPpnXfwDgMQYRFeHeW4gsIMCKcVh36gma5dp/A6AG+yjxFrK4OBWr2MMuI79W+oBl2oXKPwLgMQYRFWHzFFwQYEZ5UhI9DdSS+gAAjwukiXG2CVcqsjt/iF+4FEGBEOOcIlxjf2j0AAowI5xxh8QUEGBHeQ4Rb8QV2E+AfXYW3asJQoGbLubTmAeyKISDC4gsIMCKcZXxW4gvsPMAfjgPuoAlDgZqZfv5D1zTkwRAQ4fl87ta1DAKMCOceYfEFBBgRThzhWnwBAUaE00b4OL59QZ+rc81CtgG+AncrYnVlKNBh6Av6PBHfK5XrFfJkCIhwmcQXsg/wD6/AeTVhIDtdqFyfkDdDQITL0ofadQkCjAgjvoAAI8LiCwgwIszuXHD9gQAjwoKYVuO6AwEGERZfQIARYfEFBBgRZndWri8QYBDhtFrXFQgwiLD4AucK8A+uQApNGNhK6zqCchx8KA5IpAkD59KFyjUE5YjjMqQUEb48cFcivpcr1w6UxRAQYfEFBBgR5oRefEGAQYTTx7d2jYAAgwiLLyDAiLD4AgIMIrw7S9cCCDCIcFqNawDmFODvX4acNGGYocbuYV4MAREWX0CAYXYRbu0aBBhEWHwBAYaiIyy+IMCGgAgntrZTwBAQ4bS6UNknEMclmIqI8KVhwiK+lyp7BI4ZAiIsvsA+AvzBOGBimjBMSB8W9gacZAiI8Pjxre0LeHuAv3cJpqoJQ8b6UNsTcDuGgAiLLyDAUEyEG3sBBBgRFl9AgKHoCIsvIMCIcGKH5g8IMCKcVmvugABD2giLLyDA8JYI9yPHd2XOgADD2x3/LW430t/5Ls0XOH+Av3sJ5uAw9GHYgTZUZgpsI46LMBcRzYsR4osR4ovDOUR4Ly7MEdgFQ2CulmEV1ncIbreJbrOJt7kBAgwjviWbAyDAACDAAIAAA8BkA/yBOACAtAwBAAQYAAQYABgtwN+5CAAkZggAIMAAIMAAgAADQFEBfgwASMwQAECAAUCAAQABBoCSAvztxwCAxAwBAAQYAAQYABgrwO+PAwBIyxAAQIABQIABAAEGAAEGALYK8LeWAEBihgAAAgwAAgwACDAACDAAIMAAIMAAgAADgAADAAIMAPsK8DeXAEBiB++LAwBIyxAAQIABYDYBfhQASMwQAECAAUCAAQABBgABBgC2CvCNRwGAxAwBAAQYAAQYABBgABBgAECAAUCAAQABBoAsA/wIAJDYwT1xAABpHdzzjUcAgMQMAQAEGAAEGAAQYAAQYABAgAFAgAEAAQYAAQYA3gjw1x8BABIzBAAQYACYTYAfBgASMwQAEGAAEGAAYKwAvzcOACAtQwAAAQaAuQT4aw8DAIkZAgAIMAAIMAAgwAAgwACAAAOAAAMA7x7ghwCAxAwBAAQYAOYS4K8+BAAkZggAIMAAIMAAgAADQEEBfk8cAEBahgAAAgwAAgwACDAAlBTgrzwEACRmCACwnwA/CAAkZggAIMAAIMAAgAADgAADAAIMAAIMAAgwAOQY4C8/CAAkZggAsI8AL+IAANIyBAAQYAAQYABgvAA/AAAkZggAIMAAIMAAwGgB/tIDAEBihgAAAgwAAgwACDAACDAAIMAAIMAAgAADgAADAAIMAPvxP1CjqIdWA1T4AAAAAElFTkSuQmCC");
	background-size: 168rpx 168rpx;
	background-repeat: no-repeat;
	background-position: left top;
	animation: wipeIn .45s .85s cubic-bezier(.4, 0, .3, 1) both;
}
@keyframes wipeIn {
	from { width: 0; }
	to   { width: 168rpx; }
}
.success-text {
	margin-top: 28rpx; font-size: 32rpx; font-weight: 700; color: #248a3d;
	opacity: 0; transform: translateY(12rpx);
	animation: textUp .3s .75s cubic-bezier(.34,1.56,.64,1) forwards;
}
@keyframes textUp { to { opacity: 1; transform: none; } }
</style>
