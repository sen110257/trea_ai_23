<template>
  <div class="game-container" ref="gameContainer">
    <canvas ref="gameCanvas" class="game-canvas"></canvas>
    
    <!-- UI覆盖层 -->
    <div class="ui-overlay">
      <!-- 分数显示 -->
      <div class="score-panel" v-if="gameState !== 'idle'">
        <div class="score-item">
          <span class="label">得分</span>
          <span class="value">{{ score }}</span>
        </div>
        <div class="score-item">
          <span class="label">时间</span>
          <span class="value">{{ formatTime(survivalTime) }}</span>
        </div>
        <div class="score-item">
          <span class="label">最高</span>
          <span class="value">{{ highScore }}</span>
        </div>
      </div>

      <!-- 道具状态 -->
      <div class="powerup-status" v-if="gameState === 'playing'">
        <div class="powerup-item" v-if="hasShield">
          <span class="shield-icon">🛡️</span>
          <span class="powerup-time">{{ formatTime(shieldTime) }}</span>
        </div>
        <div class="powerup-item" v-if="isSlowMode">
          <span class="slow-icon">⏱️</span>
          <span class="powerup-time">{{ formatTime(slowTime) }}</span>
        </div>
      </div>
    </div>

    <!-- 开始界面 -->
    <div class="menu-overlay" v-if="gameState === 'idle'">
      <div class="menu-content">
        <h1 class="game-title">躲避线条</h1>
        <p class="game-subtitle">操控小点躲避移动的线条障碍</p>
        <div class="controls-hint">
          <p>键盘：方向键 / WASD</p>
          <p>鼠标：按住拖拽控制</p>
          <p>触屏：滑动控制</p>
        </div>
        <button class="game-btn start-btn" @click="startGame">
          开始游戏
        </button>
        <div class="high-score-display" v-if="highScore > 0">
          历史最高分：{{ highScore }}
        </div>
      </div>
    </div>

    <!-- 暂停界面 -->
    <div class="menu-overlay" v-if="gameState === 'paused'">
      <div class="menu-content">
        <h2 class="game-title">游戏暂停</h2>
        <div class="current-score">
          得分：{{ score }} | 时间：{{ formatTime(survivalTime) }}
        </div>
        <div class="btn-group">
          <button class="game-btn resume-btn" @click="resumeGame">
            继续游戏
          </button>
          <button class="game-btn restart-btn" @click="restartGame">
            重新开始
          </button>
        </div>
      </div>
    </div>

    <!-- 游戏结束界面 -->
    <div class="menu-overlay" v-if="gameState === 'gameover'">
      <div class="menu-content">
        <h2 class="game-title gameover-title">游戏结束</h2>
        <div class="final-stats">
          <div class="stat-item">
            <span class="stat-label">得分</span>
            <span class="stat-value">{{ score }}</span>
          </div>
          <div class="stat-item">
            <span class="stat-label">存活时间</span>
            <span class="stat-value">{{ formatTime(survivalTime) }}</span>
          </div>
          <div class="stat-item" v-if="score >= highScore && score > 0">
            <span class="new-record">🎉 新纪录！</span>
          </div>
        </div>
        <div class="btn-group">
          <button class="game-btn start-btn" @click="restartGame">
            再来一局
          </button>
          <button class="game-btn menu-btn" @click="goToMenu">
            返回菜单
          </button>
        </div>
        <div class="high-score-display">
          历史最高分：{{ highScore }}
        </div>
      </div>
    </div>

    <!-- 暂停按钮 -->
    <button 
      class="pause-btn" 
      v-if="gameState === 'playing'"
      @click="pauseGame"
    >
      ⏸️
    </button>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

// 游戏状态
const gameCanvas = ref(null)
const gameContainer = ref(null)
const gameState = ref('idle') // idle, playing, paused, gameover

// 游戏数据
const score = ref(0)
const survivalTime = ref(0)
const highScore = ref(0)

// 道具状态
const hasShield = ref(false)
const shieldTime = ref(0)
const isSlowMode = ref(false)
const slowTime = ref(0)

// Canvas上下文
let ctx = null
let canvasWidth = 0
let canvasHeight = 0
let animationId = null
let lastTime = 0
let gameStartTime = 0
let difficultyMultiplier = 1

// 玩家
const player = {
  x: 0,
  y: 0,
  radius: 12,
  speed: 6,
  color: '#00ffff',
  glowColor: 'rgba(0, 255, 255, 0.5)'
}

// 输入状态
const keys = {
  up: false,
  down: false,
  left: false,
  right: false
}

const touch = {
  active: false,
  startX: 0,
  startY: 0,
  currentX: 0,
  currentY: 0
}

// 鼠标状态
const mouse = {
  active: false,
  startX: 0,
  startY: 0,
  currentX: 0,
  currentY: 0
}

// 障碍物数组
let obstacles = []
let powerups = []
let particles = []

// 障碍物类型
const OBSTACLE_TYPES = {
  HORIZONTAL: 'horizontal',
  VERTICAL: 'vertical',
  DIAGONAL: 'diagonal'
}

// 道具类型
const POWERUP_TYPES = {
  SHIELD: 'shield',
  SLOW: 'slow'
}

// 格式化时间
const formatTime = (seconds) => {
  const mins = Math.floor(seconds / 60)
  const secs = Math.floor(seconds % 60)
  return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`
}

// 从本地存储加载最高分
const loadHighScore = () => {
  const saved = localStorage.getItem('dodgeGame_highScore')
  if (saved) {
    highScore.value = parseInt(saved)
  }
}

// 保存最高分
const saveHighScore = () => {
  if (score.value > highScore.value) {
    highScore.value = score.value
    localStorage.setItem('dodgeGame_highScore', highScore.value.toString())
  }
}

// 初始化Canvas
const initCanvas = () => {
  const container = gameContainer.value
  const canvas = gameCanvas.value
  
  if (!container || !canvas) return
  
  canvasWidth = container.clientWidth
  canvasHeight = container.clientHeight
  
  canvas.width = canvasWidth
  canvas.height = canvasHeight
  
  ctx = canvas.getContext('2d')
  
  // 重置玩家位置到中心
  player.x = canvasWidth / 2
  player.y = canvasHeight / 2
}

// 创建障碍物
const createObstacle = () => {
  const types = Object.values(OBSTACLE_TYPES)
  const type = types[Math.floor(Math.random() * types.length)]
  
  // 初始速度更低，递增更平滑
  // 初始速度: 1.5，每20秒增加0.3倍，上限5倍
  const baseSpeed = 1.5 + Math.min(difficultyMultiplier, 5) * 0.5
  const speed = isSlowMode.value ? baseSpeed * 0.5 : baseSpeed
  
  // 初始线条更短，随难度递增变长
  // 初始长度: 80-150，随难度增加到150-300
  const lengthMultiplier = 1 + Math.min(difficultyMultiplier, 3) * 0.5
  
  let obstacle = {
    type,
    speed,
    color: `hsl(${Math.random() * 60 + 180}, 100%, 60%)`,
    width: 4,
    glowColor: 'rgba(0, 200, 255, 0.3)'
  }
  
  switch (type) {
    case OBSTACLE_TYPES.HORIZONTAL:
      obstacle.y = Math.random() * canvasHeight
      obstacle.length = (Math.random() * 120 + 80) * lengthMultiplier
      obstacle.x = -obstacle.length
      obstacle.vx = speed
      obstacle.vy = 0
      break
      
    case OBSTACLE_TYPES.VERTICAL:
      obstacle.x = Math.random() * canvasWidth
      obstacle.length = (Math.random() * 120 + 80) * lengthMultiplier
      obstacle.y = -obstacle.length
      obstacle.vx = 0
      obstacle.vy = speed
      break
      
    case OBSTACLE_TYPES.DIAGONAL:
      const fromLeft = Math.random() > 0.5
      obstacle.length = (Math.random() * 100 + 60) * lengthMultiplier
      obstacle.y = -obstacle.length
      obstacle.x = fromLeft ? -obstacle.length : canvasWidth + obstacle.length
      obstacle.vx = fromLeft ? speed * 0.7 : -speed * 0.7
      obstacle.vy = speed
      break
  }
  
  obstacles.push(obstacle)
}

// 创建道具
const createPowerup = () => {
  if (Math.random() > 0.3) return // 70%概率生成
  
  const types = Object.values(POWERUP_TYPES)
  const type = types[Math.floor(Math.random() * types.length)]
  
  const powerup = {
    type,
    x: Math.random() * (canvasWidth - 60) + 30,
    y: Math.random() * (canvasHeight - 60) + 30,
    radius: 15,
    pulsePhase: 0
  }
  
  powerups.push(powerup)
}

// 创建粒子效果
const createParticles = (x, y, color, count = 10) => {
  for (let i = 0; i < count; i++) {
    particles.push({
      x,
      y,
      vx: (Math.random() - 0.5) * 8,
      vy: (Math.random() - 0.5) * 8,
      radius: Math.random() * 4 + 2,
      color,
      life: 1,
      decay: Math.random() * 0.03 + 0.02
    })
  }
}

// 更新玩家位置
const updatePlayer = () => {
  let dx = 0
  let dy = 0
  
  // 鼠标拖拽控制（最高优先级，实时跟随）
  if (mouse.active) {
    // 计算鼠标相对于Canvas的坐标
    const canvas = gameCanvas.value
    const rect = canvas.getBoundingClientRect()
    const mouseCanvasX = mouse.currentX - rect.left
    const mouseCanvasY = mouse.currentY - rect.top
    
    // 直接移动到鼠标位置，使用平滑插值
    const targetX = mouseCanvasX
    const targetY = mouseCanvasY
    
    // 计算方向向量
    const dirX = targetX - player.x
    const dirY = targetY - player.y
    const distance = Math.sqrt(dirX * dirX + dirY * dirY)
    
    // 如果距离很小，直接定位
    if (distance < 2) {
      player.x = targetX
      player.y = targetY
    } else {
      // 平滑移动，速度随距离增加
      const moveSpeed = Math.min(player.speed * 1.5, distance * 0.15)
      const normalizedX = dirX / distance
      const normalizedY = dirY / distance
      dx = normalizedX * moveSpeed
      dy = normalizedY * moveSpeed
    }
  }
  // 键盘控制
  else if (keys.up || keys.down || keys.left || keys.right) {
    if (keys.up) dy -= player.speed
    if (keys.down) dy += player.speed
    if (keys.left) dx -= player.speed
    if (keys.right) dx += player.speed
  }
  // 触屏控制
  else if (touch.active) {
    const touchDx = touch.currentX - touch.startX
    const touchDy = touch.currentY - touch.startY
    const distance = Math.sqrt(touchDx * touchDx + touchDy * touchDy)
    
    if (distance > 10) {
      const normalizedX = touchDx / distance
      const normalizedY = touchDy / distance
      dx = normalizedX * player.speed
      dy = normalizedY * player.speed
    }
  }
  
  // 限制对角线速度（仅键盘和触屏）
  if ((keys.up || keys.down || keys.left || keys.right || touch.active) && dx !== 0 && dy !== 0) {
    const factor = 1 / Math.sqrt(2)
    dx *= factor
    dy *= factor
  }
  
  // 更新位置并限制在边界内
  player.x = Math.max(player.radius, Math.min(canvasWidth - player.radius, player.x + dx))
  player.y = Math.max(player.radius, Math.min(canvasHeight - player.radius, player.y + dy))
}

// 更新障碍物
const updateObstacles = () => {
  for (let i = obstacles.length - 1; i >= 0; i--) {
    const obs = obstacles[i]
    obs.x += obs.vx
    obs.y += obs.vy
    
    // 移除屏幕外的障碍物
    const isOffScreen = 
      obs.x > canvasWidth + obs.length + 50 ||
      obs.x < -obs.length - 50 ||
      obs.y > canvasHeight + obs.length + 50 ||
      obs.y < -obs.length - 50
    
    if (isOffScreen) {
      obstacles.splice(i, 1)
    }
  }
}

// 更新道具
const updatePowerups = () => {
  for (let i = powerups.length - 1; i >= 0; i--) {
    powerups[i].pulsePhase += 0.1
  }
}

// 更新粒子
const updateParticles = () => {
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i]
    p.x += p.vx
    p.y += p.vy
    p.life -= p.decay
    p.vx *= 0.98
    p.vy *= 0.98
    
    if (p.life <= 0) {
      particles.splice(i, 1)
    }
  }
}

// 检测碰撞：玩家与线段
const checkLineCollision = (player, obs) => {
  const px = player.x
  const py = player.y
  const r = player.radius
  
  let x1, y1, x2, y2
  
  switch (obs.type) {
    case OBSTACLE_TYPES.HORIZONTAL:
      x1 = obs.x
      y1 = obs.y
      x2 = obs.x + obs.length
      y2 = obs.y
      break
      
    case OBSTACLE_TYPES.VERTICAL:
      x1 = obs.x
      y1 = obs.y
      x2 = obs.x
      y2 = obs.y + obs.length
      break
      
    case OBSTACLE_TYPES.DIAGONAL:
      // 对角线线段
      const angle = Math.atan2(obs.vy, obs.vx)
      x1 = obs.x
      y1 = obs.y
      x2 = obs.x + Math.cos(angle + Math.PI) * obs.length
      y2 = obs.y + Math.sin(angle + Math.PI) * obs.length
      break
  }
  
  // 点到线段的距离
  const A = px - x1
  const B = py - y1
  const C = x2 - x1
  const D = y2 - y1
  
  const dot = A * C + B * D
  const lenSq = C * C + D * D
  let param = -1
  
  if (lenSq !== 0) param = dot / lenSq
  
  let xx, yy
  
  if (param < 0) {
    xx = x1
    yy = y1
  } else if (param > 1) {
    xx = x2
    yy = y2
  } else {
    xx = x1 + param * C
    yy = y1 + param * D
  }
  
  const dx = px - xx
  const dy = py - yy
  const distance = Math.sqrt(dx * dx + dy * dy)
  
  return distance < r + obs.width / 2
}

// 检测碰撞：玩家与圆形道具
const checkCircleCollision = (circle1, circle2) => {
  const dx = circle1.x - circle2.x
  const dy = circle1.y - circle2.y
  const distance = Math.sqrt(dx * dx + dy * dy)
  
  return distance < circle1.radius + circle2.radius
}

// 检测所有碰撞
const checkCollisions = () => {
  // 检测与障碍物的碰撞
  for (const obs of obstacles) {
    if (checkLineCollision(player, obs)) {
      if (hasShield.value) {
        // 护盾保护，消耗护盾
        hasShield.value = false
        shieldTime.value = 0
        createParticles(player.x, player.y, '#00ff88', 20)
        // 移除碰撞的障碍物
        const index = obstacles.indexOf(obs)
        if (index > -1) obstacles.splice(index, 1)
      } else {
        // 游戏结束
        gameOver()
        return
      }
    }
  }
  
  // 检测与道具的碰撞
  for (let i = powerups.length - 1; i >= 0; i--) {
    const powerup = powerups[i]
    if (checkCircleCollision(player, powerup)) {
      // 收集道具
      switch (powerup.type) {
        case POWERUP_TYPES.SHIELD:
          hasShield.value = true
          shieldTime.value = 10
          createParticles(powerup.x, powerup.y, '#00ff88', 15)
          break
          
        case POWERUP_TYPES.SLOW:
          isSlowMode.value = true
          slowTime.value = 8
          // 降低所有现有障碍物的速度
          obstacles.forEach(obs => {
            obs.vx *= 0.5
            obs.vy *= 0.5
          })
          createParticles(powerup.x, powerup.y, '#ffaa00', 15)
          break
      }
      
      powerups.splice(i, 1)
    }
  }
}

// 更新道具时间
const updatePowerupTimers = (deltaTime) => {
  if (hasShield.value) {
    shieldTime.value -= deltaTime
    if (shieldTime.value <= 0) {
      hasShield.value = false
      shieldTime.value = 0
    }
  }
  
  if (isSlowMode.value) {
    slowTime.value -= deltaTime
    if (slowTime.value <= 0) {
      isSlowMode.value = false
      slowTime.value = 0
      // 恢复障碍物速度
      obstacles.forEach(obs => {
        obs.vx *= 2
        obs.vy *= 2
      })
    }
  }
}

// 绘制背景
const drawBackground = () => {
  // 深色渐变背景
  const gradient = ctx.createRadialGradient(
    canvasWidth / 2, canvasHeight / 2, 0,
    canvasWidth / 2, canvasHeight / 2, canvasWidth
  )
  gradient.addColorStop(0, '#1a1a2e')
  gradient.addColorStop(0.5, '#0f0f1a')
  gradient.addColorStop(1, '#0a0a0a')
  
  ctx.fillStyle = gradient
  ctx.fillRect(0, 0, canvasWidth, canvasHeight)
  
  // 微妙的网格背景
  ctx.strokeStyle = 'rgba(255, 255, 255, 0.03)'
  ctx.lineWidth = 1
  
  const gridSize = 40
  for (let x = 0; x < canvasWidth; x += gridSize) {
    ctx.beginPath()
    ctx.moveTo(x, 0)
    ctx.lineTo(x, canvasHeight)
    ctx.stroke()
  }
  
  for (let y = 0; y < canvasHeight; y += gridSize) {
    ctx.beginPath()
    ctx.moveTo(0, y)
    ctx.lineTo(canvasWidth, y)
    ctx.stroke()
  }
}

// 绘制玩家
const drawPlayer = () => {
  // 发光效果
  ctx.save()
  ctx.shadowColor = hasShield.value ? '#00ff88' : player.glowColor
  ctx.shadowBlur = hasShield.value ? 25 : 15
  
  // 护盾效果
  if (hasShield.value) {
    ctx.beginPath()
    ctx.arc(player.x, player.y, player.radius + 8, 0, Math.PI * 2)
    ctx.strokeStyle = `rgba(0, 255, 136, ${0.5 + Math.sin(Date.now() / 200) * 0.3})`
    ctx.lineWidth = 2
    ctx.stroke()
  }
  
  // 玩家主体
  ctx.beginPath()
  ctx.arc(player.x, player.y, player.radius, 0, Math.PI * 2)
  ctx.fillStyle = hasShield.value ? '#00ff88' : player.color
  ctx.fill()
  
  ctx.restore()
  
  // 内部高光
  ctx.beginPath()
  ctx.arc(player.x - 3, player.y - 3, player.radius * 0.4, 0, Math.PI * 2)
  ctx.fillStyle = 'rgba(255, 255, 255, 0.4)'
  ctx.fill()
}

// 绘制障碍物
const drawObstacles = () => {
  for (const obs of obstacles) {
    ctx.save()
    ctx.shadowColor = obs.glowColor
    ctx.shadowBlur = 10
    
    ctx.strokeStyle = isSlowMode.value ? '#ffaa00' : obs.color
    ctx.lineWidth = obs.width
    ctx.lineCap = 'round'
    
    ctx.beginPath()
    
    switch (obs.type) {
      case OBSTACLE_TYPES.HORIZONTAL:
        ctx.moveTo(obs.x, obs.y)
        ctx.lineTo(obs.x + obs.length, obs.y)
        break
        
      case OBSTACLE_TYPES.VERTICAL:
        ctx.moveTo(obs.x, obs.y)
        ctx.lineTo(obs.x, obs.y + obs.length)
        break
        
      case OBSTACLE_TYPES.DIAGONAL:
        const angle = Math.atan2(obs.vy, obs.vx)
        ctx.moveTo(obs.x, obs.y)
        ctx.lineTo(
          obs.x + Math.cos(angle + Math.PI) * obs.length,
          obs.y + Math.sin(angle + Math.PI) * obs.length
        )
        break
    }
    
    ctx.stroke()
    ctx.restore()
  }
}

// 绘制道具
const drawPowerups = () => {
  for (const powerup of powerups) {
    ctx.save()
    
    const pulse = 1 + Math.sin(powerup.pulsePhase) * 0.2
    const radius = powerup.radius * pulse
    
    // 发光效果
    const glowColor = powerup.type === POWERUP_TYPES.SHIELD 
      ? 'rgba(0, 255, 136, 0.4)' 
      : 'rgba(255, 170, 0, 0.4)'
    
    ctx.shadowColor = glowColor
    ctx.shadowBlur = 20
    
    // 外圆
    ctx.beginPath()
    ctx.arc(powerup.x, powerup.y, radius, 0, Math.PI * 2)
    ctx.fillStyle = powerup.type === POWERUP_TYPES.SHIELD 
      ? 'rgba(0, 255, 136, 0.3)' 
      : 'rgba(255, 170, 0, 0.3)'
    ctx.fill()
    
    // 图标
    ctx.font = `${radius * 1.2}px Arial`
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText(
      powerup.type === POWERUP_TYPES.SHIELD ? '🛡️' : '⏱️',
      powerup.x,
      powerup.y
    )
    
    ctx.restore()
  }
}

// 绘制粒子
const drawParticles = () => {
  for (const p of particles) {
    ctx.save()
    ctx.globalAlpha = p.life
    ctx.beginPath()
    ctx.arc(p.x, p.y, p.radius * p.life, 0, Math.PI * 2)
    ctx.fillStyle = p.color
    ctx.fill()
    ctx.restore()
  }
}

// 游戏主循环
const gameLoop = (timestamp) => {
  if (gameState.value !== 'playing') return
  
  const deltaTime = (timestamp - lastTime) / 1000
  lastTime = timestamp
  
  // 更新游戏时间和难度
  survivalTime.value += deltaTime
  score.value = Math.floor(survivalTime.value * 10)
  
  // 难度平滑递增：每20秒增加0.3倍，上限5倍
  // 初始难度非常低，给新手适应时间
  difficultyMultiplier = Math.min(Math.floor(survivalTime.value / 20) * 0.3, 5)
  
  // 随机生成障碍物
  // 初始生成概率极低(0.5%)，随难度递增最高到5%
  // 同时，存活时间越长，允许同时存在的障碍物越多
  const baseSpawnChance = 0.005
  const maxSpawnChance = 0.05
  const spawnChance = Math.min(baseSpawnChance + difficultyMultiplier * 0.009, maxSpawnChance)
  
  // 限制同时存在的障碍物数量，避免过度拥挤
  // 初始上限5个，每20秒增加2个，上限20个
  const maxObstacles = Math.min(5 + Math.floor(difficultyMultiplier * 4), 20)
  
  if (Math.random() < spawnChance && obstacles.length < maxObstacles) {
    createObstacle()
  }
  
  // 随机生成道具
  // 初始概率较高(0.8%)，增加趣味性
  // 道具数量限制：最多同时存在3个
  const powerupChance = 0.008
  const maxPowerups = 3
  
  if (Math.random() < powerupChance && powerups.length < maxPowerups) {
    createPowerup()
  }
  
  // 更新
  updatePlayer()
  updateObstacles()
  updatePowerups()
  updateParticles()
  updatePowerupTimers(deltaTime)
  checkCollisions()
  
  // 绘制
  drawBackground()
  drawParticles()
  drawObstacles()
  drawPowerups()
  drawPlayer()
  
  animationId = requestAnimationFrame(gameLoop)
}

// 游戏控制函数
const startGame = () => {
  resetGame()
  gameState.value = 'playing'
  gameStartTime = Date.now()
  lastTime = performance.now()
  animationId = requestAnimationFrame(gameLoop)
}

const pauseGame = () => {
  if (gameState.value !== 'playing') return
  gameState.value = 'paused'
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
}

const resumeGame = () => {
  if (gameState.value !== 'paused') return
  gameState.value = 'playing'
  lastTime = performance.now()
  animationId = requestAnimationFrame(gameLoop)
}

const restartGame = () => {
  startGame()
}

const goToMenu = () => {
  resetGame()
  gameState.value = 'idle'
}

const gameOver = () => {
  gameState.value = 'gameover'
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
  saveHighScore()
  createParticles(player.x, player.y, '#ff4444', 30)
  
  // 绘制最后一帧
  drawBackground()
  drawParticles()
  drawObstacles()
  drawPowerups()
}

const resetGame = () => {
  obstacles = []
  powerups = []
  particles = []
  score.value = 0
  survivalTime.value = 0
  difficultyMultiplier = 1
  hasShield.value = false
  shieldTime.value = 0
  isSlowMode.value = false
  slowTime.value = 0
  
  player.x = canvasWidth / 2
  player.y = canvasHeight / 2
}

// 键盘事件处理
const handleKeyDown = (e) => {
  switch (e.key) {
    case 'ArrowUp':
    case 'w':
    case 'W':
      keys.up = true
      break
    case 'ArrowDown':
    case 's':
    case 'S':
      keys.down = true
      break
    case 'ArrowLeft':
    case 'a':
    case 'A':
      keys.left = true
      break
    case 'ArrowRight':
    case 'd':
    case 'D':
      keys.right = true
      break
    case ' ':
    case 'Escape':
      if (gameState.value === 'playing') {
        pauseGame()
      } else if (gameState.value === 'paused') {
        resumeGame()
      }
      break
  }
}

const handleKeyUp = (e) => {
  switch (e.key) {
    case 'ArrowUp':
    case 'w':
    case 'W':
      keys.up = false
      break
    case 'ArrowDown':
    case 's':
    case 'S':
      keys.down = false
      break
    case 'ArrowLeft':
    case 'a':
    case 'A':
      keys.left = false
      break
    case 'ArrowRight':
    case 'd':
    case 'D':
      keys.right = false
      break
  }
}

// 触屏事件处理
const handleTouchStart = (e) => {
  if (gameState.value !== 'playing') return
  
  const touchEvent = e.touches[0]
  touch.active = true
  touch.startX = touchEvent.clientX
  touch.startY = touchEvent.clientY
  touch.currentX = touchEvent.clientX
  touch.currentY = touchEvent.clientY
}

const handleTouchMove = (e) => {
  if (!touch.active || gameState.value !== 'playing') return
  
  e.preventDefault()
  const touchEvent = e.touches[0]
  touch.currentX = touchEvent.clientX
  touch.currentY = touchEvent.clientY
}

const handleTouchEnd = () => {
  touch.active = false
}

// 鼠标事件处理
const handleMouseDown = (e) => {
  if (gameState.value !== 'playing') return
  
  mouse.active = true
  mouse.startX = e.clientX
  mouse.startY = e.clientY
  mouse.currentX = e.clientX
  mouse.currentY = e.clientY
}

const handleMouseMove = (e) => {
  if (!mouse.active || gameState.value !== 'playing') return
  
  mouse.currentX = e.clientX
  mouse.currentY = e.clientY
}

const handleMouseUp = () => {
  mouse.active = false
}

const handleMouseLeave = () => {
  mouse.active = false
}

// 窗口大小变化处理
const handleResize = () => {
  initCanvas()
}

// 生命周期
onMounted(() => {
  loadHighScore()
  initCanvas()
  
  // 绘制初始背景
  drawBackground()
  
  // 添加事件监听
  window.addEventListener('keydown', handleKeyDown)
  window.addEventListener('keyup', handleKeyUp)
  window.addEventListener('resize', handleResize)
  
  const container = gameContainer.value
  const canvas = gameCanvas.value
  if (container) {
    container.addEventListener('touchstart', handleTouchStart, { passive: false })
    container.addEventListener('touchmove', handleTouchMove, { passive: false })
    container.addEventListener('touchend', handleTouchEnd)
  }
  
  // 添加鼠标事件监听
  if (canvas) {
    canvas.addEventListener('mousedown', handleMouseDown)
    canvas.addEventListener('mousemove', handleMouseMove)
    canvas.addEventListener('mouseup', handleMouseUp)
    canvas.addEventListener('mouseleave', handleMouseLeave)
  }
})

onUnmounted(() => {
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
  
  // 移除事件监听
  window.removeEventListener('keydown', handleKeyDown)
  window.removeEventListener('keyup', handleKeyUp)
  window.removeEventListener('resize', handleResize)
  
  const container = gameContainer.value
  const canvas = gameCanvas.value
  if (container) {
    container.removeEventListener('touchstart', handleTouchStart)
    container.removeEventListener('touchmove', handleTouchMove)
    container.removeEventListener('touchend', handleTouchEnd)
  }
  
  // 移除鼠标事件监听
  if (canvas) {
    canvas.removeEventListener('mousedown', handleMouseDown)
    canvas.removeEventListener('mousemove', handleMouseMove)
    canvas.removeEventListener('mouseup', handleMouseUp)
    canvas.removeEventListener('mouseleave', handleMouseLeave)
  }
})
</script>

<style scoped>
.game-container {
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden;
  background: linear-gradient(135deg, #0a0a0a 0%, #1a1a2e 50%, #16213e 100%);
}

.game-canvas {
  display: block;
  width: 100%;
  height: 100%;
}

.ui-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  pointer-events: none;
  z-index: 10;
}

.score-panel {
  display: flex;
  justify-content: space-around;
  padding: 15px;
  background: rgba(0, 0, 0, 0.5);
  backdrop-filter: blur(10px);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.score-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  color: #fff;
}

.score-item .label {
  font-size: 12px;
  opacity: 0.7;
  margin-bottom: 3px;
}

.score-item .value {
  font-size: 20px;
  font-weight: bold;
  color: #00ffff;
  text-shadow: 0 0 10px rgba(0, 255, 255, 0.5);
}

.powerup-status {
  position: absolute;
  top: 80px;
  left: 15px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.powerup-item {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  background: rgba(0, 0, 0, 0.6);
  backdrop-filter: blur(10px);
  border-radius: 20px;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.powerup-item span {
  font-size: 16px;
}

.powerup-time {
  color: #fff;
  font-size: 14px;
  font-weight: bold;
}

.menu-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(0, 0, 0, 0.8);
  backdrop-filter: blur(15px);
  z-index: 20;
}

.menu-content {
  text-align: center;
  padding: 40px;
  background: rgba(20, 20, 40, 0.9);
  border-radius: 30px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.5);
  max-width: 90vw;
}

.game-title {
  font-size: 42px;
  font-weight: bold;
  color: #fff;
  margin-bottom: 15px;
  text-shadow: 0 0 30px rgba(0, 255, 255, 0.5);
  background: linear-gradient(90deg, #00ffff, #00ff88);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.gameover-title {
  background: linear-gradient(90deg, #ff4444, #ff8844);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.game-subtitle {
  font-size: 16px;
  color: rgba(255, 255, 255, 0.7);
  margin-bottom: 25px;
}

.controls-hint {
  font-size: 14px;
  color: rgba(255, 255, 255, 0.5);
  margin-bottom: 30px;
  line-height: 1.8;
}

.controls-hint p {
  margin: 5px 0;
}

.btn-group {
  display: flex;
  flex-direction: column;
  gap: 15px;
  margin-bottom: 20px;
}

@media (min-width: 768px) {
  .btn-group {
    flex-direction: row;
    justify-content: center;
  }
}

.game-btn {
  padding: 15px 40px;
  font-size: 16px;
  font-weight: bold;
  border: none;
  border-radius: 25px;
  cursor: pointer;
  transition: all 0.3s ease;
  backdrop-filter: blur(10px);
  text-transform: uppercase;
  letter-spacing: 1px;
}

.start-btn {
  background: linear-gradient(135deg, rgba(0, 255, 255, 0.3), rgba(0, 255, 136, 0.3));
  border: 2px solid rgba(0, 255, 255, 0.5);
  color: #00ffff;
}

.start-btn:hover {
  background: linear-gradient(135deg, rgba(0, 255, 255, 0.5), rgba(0, 255, 136, 0.5));
  transform: scale(1.05);
  box-shadow: 0 0 30px rgba(0, 255, 255, 0.4);
}

.resume-btn {
  background: linear-gradient(135deg, rgba(0, 255, 136, 0.3), rgba(0, 200, 100, 0.3));
  border: 2px solid rgba(0, 255, 136, 0.5);
  color: #00ff88;
}

.resume-btn:hover {
  background: linear-gradient(135deg, rgba(0, 255, 136, 0.5), rgba(0, 200, 100, 0.5));
  transform: scale(1.05);
  box-shadow: 0 0 30px rgba(0, 255, 136, 0.4);
}

.restart-btn {
  background: linear-gradient(135deg, rgba(255, 136, 0, 0.3), rgba(255, 100, 0, 0.3));
  border: 2px solid rgba(255, 170, 0, 0.5);
  color: #ffaa00;
}

.restart-btn:hover {
  background: linear-gradient(135deg, rgba(255, 136, 0, 0.5), rgba(255, 100, 0, 0.5));
  transform: scale(1.05);
  box-shadow: 0 0 30px rgba(255, 170, 0, 0.4);
}

.menu-btn {
  background: linear-gradient(135deg, rgba(255, 255, 255, 0.1), rgba(200, 200, 200, 0.1));
  border: 2px solid rgba(255, 255, 255, 0.3);
  color: rgba(255, 255, 255, 0.8);
}

.menu-btn:hover {
  background: linear-gradient(135deg, rgba(255, 255, 255, 0.2), rgba(200, 200, 200, 0.2));
  transform: scale(1.05);
}

.current-score {
  font-size: 18px;
  color: rgba(255, 255, 255, 0.8);
  margin-bottom: 30px;
}

.final-stats {
  margin: 25px 0;
}

.stat-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 25px;
  margin: 10px 0;
  background: rgba(255, 255, 255, 0.05);
  border-radius: 15px;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.stat-label {
  font-size: 16px;
  color: rgba(255, 255, 255, 0.7);
}

.stat-value {
  font-size: 24px;
  font-weight: bold;
  color: #00ffff;
  text-shadow: 0 0 10px rgba(0, 255, 255, 0.5);
}

.new-record {
  width: 100%;
  text-align: center;
  font-size: 20px;
  font-weight: bold;
  color: #ffaa00;
  text-shadow: 0 0 15px rgba(255, 170, 0, 0.5);
}

.high-score-display {
  font-size: 14px;
  color: rgba(255, 255, 255, 0.5);
  margin-top: 20px;
  padding-top: 15px;
  border-top: 1px solid rgba(255, 255, 255, 0.1);
}

.pause-btn {
  position: absolute;
  top: 15px;
  right: 15px;
  width: 50px;
  height: 50px;
  border: none;
  border-radius: 50%;
  background: rgba(0, 0, 0, 0.6);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.2);
  color: #fff;
  font-size: 20px;
  cursor: pointer;
  transition: all 0.3s ease;
  z-index: 15;
}

.pause-btn:hover {
  background: rgba(0, 0, 0, 0.8);
  transform: scale(1.1);
}

/* 响应式调整 */
@media (max-width: 768px) {
  .menu-content {
    padding: 25px;
  }
  
  .game-title {
    font-size: 32px;
  }
  
  .game-subtitle {
    font-size: 14px;
  }
  
  .controls-hint {
    font-size: 12px;
  }
  
  .game-btn {
    padding: 12px 30px;
    font-size: 14px;
  }
  
  .score-item .label {
    font-size: 10px;
  }
  
  .score-item .value {
    font-size: 16px;
  }
}

/* 触摸设备优化 */
@media (hover: none) and (pointer: coarse) {
  .game-btn {
    min-height: 48px;
  }
  
  .pause-btn {
    width: 60px;
    height: 60px;
    font-size: 24px;
  }
}
</style>
