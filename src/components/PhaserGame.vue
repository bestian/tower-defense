<template>
  <div ref="gameContainer" class="phaser-container"></div>

  <!-- 建造塔按鈕 - 固定在底部 -->
  <div class="tower-buttons">
    <button @click="selectTower('melee')" :class="{ selected: selectedTower === 'melee' }">
      近程塔 (${{ towerCosts.melee }})
    </button>
    <button @click="selectTower('ranged')" :class="{ selected: selectedTower === 'ranged' }">
      遠程塔 (${{ towerCosts.ranged }})
    </button>
    <button @click="selectTower('flying')" :class="{ selected: selectedTower === 'flying' }">
      飛行塔 (${{ towerCosts.flying }})
    </button>
  </div>

  <!-- 抽屜 - 只包含遊戲資訊和控制 -->
  <div class="ui-drawer" :class="{ expanded: isDrawerExpanded }">
    <div class="drawer-handle" @click="toggleDrawer">
      <span class="handle-icon">{{ isDrawerExpanded ? '◀' : '▶' }}</span>
      <span class="handle-text">{{ isDrawerExpanded ? '收起' : '遊戲資訊' }}</span>
    </div>
    <div class="drawer-content">
      <div class="game-info">
        <div class="info-item hp">
          <span class="label">玩家血量:</span>
          <span class="value">{{ playerHp }}</span>
        </div>
        <div class="info-item money">
          <span class="label">金錢:</span>
          <span class="value">${{ money }}</span>
        </div>
        <div class="info-item wave">
          <span class="label">波數:</span>
          <span class="value">{{ waveNumber }}</span>
        </div>
      </div>
      <div class="game-controls">
        <h3>遊戲控制</h3>
        <button @click="togglePause">{{ isPaused ? '繼續' : '暫停' }}</button>
        <button @click="restartGame">重新開始</button>
      </div>
      <div v-if="isGameOver" class="game-over">GAME OVER</div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'
import Phaser from 'phaser'

const gameContainer = ref<HTMLDivElement | null>(null)
let game: Phaser.Game | null = null

// UI 狀態
const money = ref(1000)
const waveNumber = ref(1)
const selectedTower = ref<string | null>(null)
const playerHp = ref(100)
const isGameOver = ref(false)
const isPaused = ref(false)
const isDrawerExpanded = ref(false)

function restartGame() {
  // 重置 UI 狀態
  isGameOver.value = false
  isPaused.value = false
  playerHp.value = 100
  money.value = 1000
  waveNumber.value = 1
  selectedTower.value = null

  // 重新啟動 Phaser 場景
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  const scene = game?.scene.getScene('GameScene') as any
  if (scene) {
    // 先停止當前場景
    scene.scene.stop()
    // 重新啟動場景
    scene.scene.start('GameScene')
  }
}

function toggleDrawer() {
  isDrawerExpanded.value = !isDrawerExpanded.value
}

function togglePause() {
  isPaused.value = !isPaused.value
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  const scene = game?.scene.getScene('GameScene') as any
  if (isPaused.value) {
    scene?.scene.pause()
  } else {
    scene?.scene.resume()
  }
}

const towerCosts = {
  melee: 100,
  ranged: 150,
  flying: 200
}

// 塔的基礎類別
class Tower extends Phaser.GameObjects.Sprite {
  protected range: number = 0
  protected damage: number = 0
  protected attackSpeed: number = 0
  protected lastAttackTime: number = 0
  protected enemies: Enemy[] = []
  protected towerType: string
  private flyingAttackTimer: number = 0; // 新增飛行塔攻擊計時器
  private rangeCircle: Phaser.GameObjects.Graphics | null = null
  // 新增塔的血量系統
  private health: number = 100
  private maxHealth: number = 100
  private healthBar: Phaser.GameObjects.Graphics | null = null
  private isDestroyed: boolean = false

    constructor(scene: Phaser.Scene, x: number, y: number, towerType: string) {
    super(scene, x, y, `${towerType}-tower`)

    this.towerType = towerType
    this.enemies = []

    // 根據塔類型設定屬性
    switch (towerType) {
      case 'melee':
        this.range = 80
        this.damage = 60
        this.attackSpeed = 1000 // 1秒攻擊一次
        this.maxHealth = 120
        this.health = 120
        break
      case 'ranged':
        this.range = 150
        this.damage = 20
        this.attackSpeed = 800 // 0.8秒攻擊一次
        this.maxHealth = 80
        this.health = 80
        break
      case 'flying':
        this.range = 100
        this.damage = 5
        this.attackSpeed = 1200 // 1.2秒攻擊一次
        this.maxHealth = 150
        this.health = 150
        break
    }

    // 設定精靈大小
    this.setScale(0.8)
    scene.add.existing(this)

    // 顯示射程圈（近程與遠程塔）
    if (towerType === 'melee' || towerType === 'ranged') {
      this.rangeCircle = scene.add.graphics()
      this.drawRangeCircle()
    }

    // 初始化血條
    this.healthBar = scene.add.graphics()
    this.updateHealthBar()
  }

  private drawRangeCircle() {
    if (!this.rangeCircle) return
    this.rangeCircle.clear()
    this.rangeCircle.fillStyle(0x3399ff, 0.18)
    this.rangeCircle.lineStyle(1, 0x3399ff, 0.4)
    this.rangeCircle.fillCircle(this.x, this.y, this.range)
    this.rangeCircle.strokeCircle(this.x, this.y, this.range)
  }

  // 新增：塔受到傷害的方法
  takeDamage(damage: number) {
    if (this.isDestroyed) return

    this.health -= damage
    this.updateHealthBar()

    // 顯示受傷效果
    this.showDamageEffect(damage)

    if (this.health <= 0) {
      this.destroy()
      this.isDestroyed = true
    }
  }

  private showDamageEffect(damage: number) {
    // 檢查場景是否存在
    if (!this.scene || !this.active) return

    // 顯示傷害數字
    const damageText = this.scene.add.text(this.x, this.y - 30, `-${damage}`, {
      fontSize: '16px',
      color: '#ff0000',
      fontStyle: 'bold'
    })

    // 傷害數字動畫
    this.scene.tweens.add({
      targets: damageText,
      y: damageText.y - 30,
      alpha: 0,
      duration: 1000,
      onComplete: () => {
        if (damageText && damageText.active) {
          damageText.destroy()
        }
      }
    })
  }

  private updateHealthBar() {
    if (!this.healthBar || this.isDestroyed) return

    this.healthBar.clear()
    const barWidth = 40
    const barHeight = 6
    const yOffset = 35

    // 血條底色
    this.healthBar.fillStyle(0x333333)
    this.healthBar.fillRect(this.x - barWidth/2, this.y - yOffset, barWidth, barHeight)

    // 血量比例
    const percent = Math.max(0, this.health / this.maxHealth)
    const healthColor = percent > 0.5 ? 0x00ff00 : percent > 0.25 ? 0xffff00 : 0xff0000
    this.healthBar.fillStyle(healthColor)
    this.healthBar.fillRect(this.x - barWidth/2, this.y - yOffset, barWidth * percent, barHeight)

    // 血條邊框
    this.healthBar.lineStyle(1, 0xffffff)
    this.healthBar.strokeRect(this.x - barWidth/2, this.y - yOffset, barWidth, barHeight)
  }

  update(time: number, enemies: Enemy[]) {
    if (this.isDestroyed) return

    this.enemies = enemies
    // 射程圈跟隨塔移動
    if (this.rangeCircle) {
      this.drawRangeCircle()
    }
    // 飛行塔自動追蹤最近敵人
    if (this.towerType === 'flying') {
      const target = this.findTarget()
      if (target) {
        const distance = Phaser.Math.Distance.Between(this.x, this.y, target.x, target.y)
        if (distance > 30) {
          // 自動移動靠近敵人
          const angle = Phaser.Math.Angle.Between(this.x, this.y, target.x, target.y)
          const speed = 2
          this.x += Math.cos(angle) * speed
          this.y += Math.sin(angle) * speed
        } else {
          // 停留或碰觸時，每100ms造成傷害
          if (time - this.flyingAttackTimer > 100) {
            target.takeDamage(this.damage)
            this.showAttackEffect(target, 0xff6600)
            this.flyingAttackTimer = time
          }
        }
      }
    } else {
      // 其他塔型照原本攻擊冷卻
      if (time - this.lastAttackTime > this.attackSpeed) {
        const target = this.findTarget()
        if (target) {
          this.attack(target, time)
        }
      }
    }
  }

  protected findTarget(): Enemy | null {
    // 尋找範圍內最近的敵人
    let closestEnemy: Enemy | null = null
    let closestDistance = this.range

    for (const enemy of this.enemies) {
      if (enemy.active) {
        const distance = Phaser.Math.Distance.Between(
          this.x, this.y,
          enemy.x, enemy.y
        )

        if (distance <= this.range && distance < closestDistance) {
          closestDistance = distance
          closestEnemy = enemy
        }
      }
    }

    return closestEnemy
  }

  protected attack(target: Enemy, time: number) {
    this.lastAttackTime = time

    // 根據塔類型執行不同攻擊
    switch (this.towerType) {
      case 'melee':
        this.meleeAttack(target)
        break
      case 'ranged':
        this.rangedAttack(target)
        break
      case 'flying':
        this.flyingAttack(target)
        break
    }
  }

  private meleeAttack(target: Enemy) {
    // 近程攻擊：直接造成傷害
    target.takeDamage(this.damage)

    // 顯示攻擊效果
    this.showAttackEffect(target, 0xff0000)
  }

    private rangedAttack(target: Enemy) {
    // 遠程攻擊：直接造成傷害
    target.takeDamage(this.damage)

    // 顯示攻擊效果
    this.showAttackEffect(target, 0x9900ff)
  }

  private flyingAttack(target: Enemy) {
    // 飛行攻擊：移動到目標附近攻擊
    const distance = Phaser.Math.Distance.Between(this.x, this.y, target.x, target.y)

    if (distance > 30) {
      // 移動靠近目標
      const angle = Phaser.Math.Angle.Between(this.x, this.y, target.x, target.y)
      const speed = 2
      this.x += Math.cos(angle) * speed
      this.y += Math.sin(angle) * speed
    } else {
      // 在攻擊範圍內，造成傷害
      target.takeDamage(this.damage)
      this.showAttackEffect(target, 0xff6600)
    }
  }

  private showAttackEffect(target: Enemy, color: number) {
    // 顯示攻擊線條效果
    const graphics = this.scene.add.graphics()
    graphics.lineStyle(3, color, 1)
    graphics.lineBetween(this.x, this.y, target.x, target.y)

    // 0.1秒後移除效果
    this.scene.time.delayedCall(100, () => {
      graphics.destroy()
    })
  }

  destroy(fromScene?: boolean): void {
    if (this.rangeCircle) {
      this.rangeCircle.destroy()
      this.rangeCircle = null
    }
    if (this.healthBar) {
      this.healthBar.destroy()
      this.healthBar = null
    }
    super.destroy(fromScene)
  }
}

// 怪物類別
class Enemy extends Phaser.GameObjects.Sprite {
  private path: Phaser.Curves.Path
  private pathProgress: number = 0
  private speed: number
  private health: number
  private maxHealth: number
  private healthBar: Phaser.GameObjects.Graphics | null = null
  // 新增敵人攻擊系統
  private attackRange: number = 30
  private attackDamage: number = 10
  private attackSpeed: number = 2000 // 2秒攻擊一次
  private lastAttackTime: number = 0
  private enemyType: string
  private targetTower: Tower | null = null

  constructor(scene: Phaser.Scene, path: Phaser.Curves.Path, health: number = 100, enemyType: string = 'normal') {
    super(scene, 0, 0, `${enemyType}-enemy`)

    this.path = path
    this.maxHealth = health
    this.health = health
    this.speed = 0.00005 // 調慢敵人速度
    this.enemyType = enemyType

    // 根據敵人類型設定屬性
    switch (enemyType) {
      case 'strong':
        this.attackDamage = 25
        this.attackSpeed = 1500
        this.attackRange = 80 // 增加攻擊範圍，和近程塔一樣
        break
      case 'fast':
        this.attackDamage = 15
        this.attackSpeed = 1000
        this.attackRange = 60 // 增加攻擊範圍
        this.speed = 0.00008 // 更快的移動速度
        break
      case 'boss':
        this.attackDamage = 50
        this.attackSpeed = 3000
        this.attackRange = 150 // 和遠程塔一樣的射程
        break
      default: // normal
        this.attackDamage = 10
        this.attackSpeed = 2000
        this.attackRange = 70 // 增加攻擊範圍
        break
    }

    // 根據敵人類型設定大小
    let scale = 1.0
    if (enemyType === 'boss') {
      scale = 1.5
    } else if (enemyType === 'strong') {
      scale = 1.3
    } else if (enemyType === 'fast') {
      scale = 0.9
    }
    this.setScale(scale)

    scene.add.existing(this)

    // 設定初始位置在路徑起點
    const startPoint = this.path.getStartPoint()
    this.setPosition(startPoint.x, startPoint.y)

    // 血條
    this.healthBar = scene.add.graphics()
    this.updateHealthBar()
  }

  update(time: number, delta: number, towers: Tower[]) {
    // 檢查是否有塔在攻擊範圍內
    this.checkForTowers(towers)

    // 如果沒有目標塔，繼續沿路徑移動
    if (!this.targetTower) {
      // 更新路徑進度
      this.pathProgress += this.speed * delta

      // 當怪物到達路徑終點時
      if (this.pathProgress >= 1) {
        // 到終點時扣玩家血量
        window.dispatchEvent(new CustomEvent('enemyReachedEnd', { detail: this.health }))
        this.destroy()
        if (this.healthBar) this.healthBar.destroy()
        return
      }

      // 根據進度計算位置
      const point = this.path.getPoint(this.pathProgress)
      this.setPosition(point.x, point.y)
      this.updateHealthBar()
    } else {
      // 有目標塔時，攻擊塔
      this.attackTower(time)
    }
  }

  private checkForTowers(towers: Tower[]) {
    if (!this.active) return

    if (this.targetTower && !this.targetTower.active) {
      this.targetTower = null
    }

    if (!this.targetTower) {
      for (const tower of towers) {
        if (tower.active) {
          const distance = Phaser.Math.Distance.Between(this.x, this.y, tower.x, tower.y)
          if (distance <= this.attackRange) {
            this.targetTower = tower
            break
          }
        }
      }
    }
  }

  private attackTower(time: number) {
    if (!this.targetTower || !this.targetTower.active || !this.active) {
      this.targetTower = null
      return
    }

    const distance = Phaser.Math.Distance.Between(this.x, this.y, this.targetTower.x, this.targetTower.y)

    if (distance <= this.attackRange) {
      // 在攻擊範圍內
      if (time - this.lastAttackTime > this.attackSpeed) {
        // 攻擊塔
        this.targetTower.takeDamage(this.attackDamage)
        this.lastAttackTime = time

        // 顯示攻擊效果
        this.showAttackEffect(this.targetTower)
      }
    } else {
      // 超出攻擊範圍，重新尋找目標
      this.targetTower = null
    }
  }

  private showAttackEffect(tower: Tower) {
    // 檢查場景是否存在
    if (!this.scene || !this.active) return

    // 顯示攻擊線條效果
    const graphics = this.scene.add.graphics()
    graphics.lineStyle(3, 0xff0000, 1)
    graphics.lineBetween(this.x, this.y, tower.x, tower.y)

    // 0.2秒後移除效果
    this.scene.time.delayedCall(200, () => {
      if (graphics && graphics.active) {
        graphics.destroy()
      }
    })
  }

  takeDamage(damage: number) {
    this.health -= damage
    if (this.health <= 0) {
      // 根據敵人類型給予不同金錢獎勵
      const baseReward = 20 + Math.floor(this.maxHealth / 10)
      let finalReward = baseReward

      switch (this.enemyType) {
        case 'strong':
          finalReward = Math.floor(baseReward * 1.5) // 強壯敵人獎勵 1.5 倍
          break
        case 'fast':
          finalReward = Math.floor(baseReward * 1.3) // 快速敵人獎勵 1.3 倍
          break
        case 'boss':
          finalReward = Math.floor(baseReward * 2.5) // 首領敵人獎勵 2.5 倍
          break
        default: // normal
          finalReward = baseReward // 普通敵人維持原獎勵
          break
      }

      window.dispatchEvent(new CustomEvent('enemyKilled', { detail: finalReward }))
      if (this.healthBar) this.healthBar.destroy()
      this.destroy()
    } else {
      this.updateHealthBar()
    }
  }

  updateHealthBar() {
    if (!this.healthBar) return
    this.healthBar.clear()
    // 動態血條高度
    const baseBarHeight = 6
    const extraBarHeight = Math.min(12, Math.floor((this.maxHealth - 100) / 100) * 2)
    const barHeight = baseBarHeight + extraBarHeight // 6~18px
    const yOffset = 28 + barHeight // 讓血條不會蓋到怪物
    // 血條底色
    this.healthBar.fillStyle(0x333333)
    this.healthBar.fillRect(this.x - 18, this.y - yOffset, 36, barHeight)
    // 血量比例
    const percent = Math.max(0, this.health / this.maxHealth)
    this.healthBar.fillStyle(0x00ff00)
    this.healthBar.fillRect(this.x - 18, this.y - yOffset, 36 * percent, barHeight)
    this.healthBar.lineStyle(1, 0xffffff)
    this.healthBar.strokeRect(this.x - 18, this.y - yOffset, 36, barHeight)
  }

  getHealthPercent(): number {
    return this.health / this.maxHealth
  }
}

// 主遊戲場景
class GameScene extends Phaser.Scene {
  private path!: Phaser.Curves.Path
  private enemies: Enemy[] = []
  private towers: Tower[] = []
  private enemySpawnTimer: number = 0
  private waveNumber: number = 1
  private enemiesInWave: number = 20
  private enemiesSpawned: number = 0
  private money: number = 1000
  private selectedTowerType: string | null = null
  private gameOver: boolean = false

  constructor() {
    super({ key: 'GameScene' })
  }

      create() {
    // 重置場景狀態
    this.enemies = []
    this.towers = []
    this.enemySpawnTimer = 0
    this.waveNumber = 1
    this.enemiesInWave = 20
    this.enemiesSpawned = 0
    this.money = 1000
    this.selectedTowerType = null
    this.gameOver = false

    // 加載圖片
    this.loadImages()

    // 響應式建立 S 型路徑
    this.createResponsiveSPath()

    // 畫出路徑（可視化用，之後可以隱藏）
    this.drawPath()

    // 敵人會在圖片加載完成後開始生成

    // 監聽 UI 事件
    window.addEventListener('enemyKilled', ((event: CustomEvent) => {
      this.money += event.detail
      window.dispatchEvent(new CustomEvent('updateMoney', { detail: this.money }))
    }) as EventListener)

    // 監聽塔選擇事件
    window.addEventListener('selectTower', ((event: CustomEvent) => {
      this.selectedTowerType = event.detail
    }) as EventListener)

    // 監聽建造塔事件
    window.addEventListener('buildTower', ((event: CustomEvent) => {
      const { x, y, type } = event.detail
      this.buildTower(x, y, type)
    }) as EventListener)

    // 設定點擊事件
    this.input.on('pointerdown', (pointer: Phaser.Input.Pointer) => {
      if (this.selectedTowerType) {
        this.buildTower(pointer.x, pointer.y, this.selectedTowerType)
      }
    })

    window.addEventListener('enemyReachedEnd', ((event: CustomEvent) => {
      if (this.gameOver) return
      const hp = event.detail
      window.dispatchEvent(new CustomEvent('playerLoseHp', { detail: hp }))
    }) as EventListener)
  }

      private loadImages() {
    // 檢查圖片是否已經加載
    if (this.textures.exists('melee-tower')) {
      // 圖片已存在，直接開始遊戲
      this.spawnEnemies()
      return
    }

    // 加載塔的 SVG 圖片（使用絕對路徑）
    this.load.image('melee-tower', 'https://bestian.github.io/tower-defense/melee-tower.svg')
    this.load.image('ranged-tower', 'https://bestian.github.io/tower-defense/ranged-tower.svg')
    this.load.image('flying-tower', 'https://bestian.github.io/tower-defense/flying-tower.svg')

    // 加載敵人的 SVG 圖片（使用絕對路徑）
    this.load.image('normal-enemy', 'https://bestian.github.io/tower-defense/normal-enemy.svg')
    this.load.image('strong-enemy', 'https://bestian.github.io/tower-defense/strong-enemy.svg')
    this.load.image('fast-enemy', 'https://bestian.github.io/tower-defense/fast-enemy.svg')
    this.load.image('boss-enemy', 'https://bestian.github.io/tower-defense/boss-enemy.svg')

    // 等待圖片加載完成
    this.load.once('complete', () => {
      // 圖片加載完成後開始遊戲
      this.spawnEnemies()
    })

    this.load.start()
  }

  private createResponsiveSPath() {
    const w = this.sys.game.config.width as number
    const h = this.sys.game.config.height as number
    const isMobile = w < 700 || h > w // 粗略判斷
    this.path = new Phaser.Curves.Path(
      w * 0.05, h * 0.15
    )
    if (isMobile) {
      // 手機：彎多一點，出口在右上
      this.path.lineTo(w * 0.25, h * 0.15)
      this.path.lineTo(w * 0.25, h * 0.35)
      this.path.lineTo(w * 0.5, h * 0.35)
      this.path.lineTo(w * 0.5, h * 0.55)
      this.path.lineTo(w * 0.75, h * 0.55)
      this.path.lineTo(w * 0.75, h * 0.75)
      this.path.lineTo(w * 0.95, h * 0.75)
    } else {
      // 電腦：出口在右上
      this.path.lineTo(w * 0.25, h * 0.15)
      this.path.lineTo(w * 0.25, h * 0.5)
      this.path.lineTo(w * 0.5, h * 0.5)
      this.path.lineTo(w * 0.5, h * 0.85)
      this.path.lineTo(w * 0.75, h * 0.85)
      this.path.lineTo(w * 0.75, h * 0.15)
      this.path.lineTo(w * 0.95, h * 0.15)
    }
  }

  private drawPath() {
    const graphics = this.add.graphics()
    graphics.lineStyle(3, 0x00ff00, 1)
    this.path.draw(graphics)
    // 在路徑起點和終點畫圓圈
    const start = this.path.getStartPoint()
    const end = this.path.getEndPoint()
    graphics.fillStyle(0x00ff00)
    graphics.fillCircle(start.x, start.y, 10)
    graphics.fillCircle(end.x, end.y, 10)
  }

    private spawnEnemies() {
    if (this.gameOver) return

    if (this.enemiesSpawned < this.enemiesInWave) {
      // 計算怪物血量（隨波數增加）
      const baseHealth = 100
      const healthIncrease = (this.waveNumber - 1) * 20
      const enemyHealth = baseHealth + healthIncrease

      // 根據波數決定敵人類型
      let enemyType = 'normal'
      const random = Math.random()

      // 從第一波開始就有機會出現所有類型
      if (this.waveNumber >= 1 && random < 0.05) {
        enemyType = 'boss' // 5% 機率出現首領
      } else if (this.waveNumber >= 1 && random < 0.15) {
        enemyType = 'strong' // 10% 機率出現強壯敵人
      } else if (this.waveNumber >= 1 && random < 0.30) {
        enemyType = 'fast' // 15% 機率出現快速敵人
      } else {
        enemyType = 'normal' // 70% 機率出現普通敵人
      }

      const enemy = new Enemy(this, this.path, enemyHealth, enemyType)
      this.enemies.push(enemy)
      this.enemiesSpawned++

      // 每 1 秒生成一隻怪物
      this.time.delayedCall(1000, () => {
        this.spawnEnemies()
      })
    }
  }

  private buildTower(x: number, y: number, type: string) {
    if (this.gameOver) return

    const costs = { melee: 100, ranged: 150, flying: 200 }
    const cost = costs[type as keyof typeof costs]

    if (this.money >= cost) {
      // 檢查是否在路徑上（簡單檢查）
      const distanceToPath = this.getDistanceToPath(x, y)
      if (distanceToPath > 30) { // 距離路徑至少 30 像素
        const tower = new Tower(this, x, y, type)
        this.towers.push(tower)
        this.money -= cost
        window.dispatchEvent(new CustomEvent('updateMoney', { detail: this.money }))
      }
    }
  }

  private getDistanceToPath(x: number, y: number): number {
    let minDistance = Infinity
    const points = this.path.getPoints(100) // 取得路徑上的 100 個點

    for (const point of points) {
      const distance = Phaser.Math.Distance.Between(x, y, point.x, point.y)
      if (distance < minDistance) {
        minDistance = distance
      }
    }

    return minDistance
  }

  update(time: number, delta: number) {
    if (this.gameOver) return
    // 更新所有怪物
    this.enemies.forEach((enemy, index) => {
      enemy.update(time, delta, this.towers)

      // 移除已銷毀的怪物
      if (!enemy.active) {
        this.enemies.splice(index, 1)
      }
    })

    // 更新所有塔
    this.towers.forEach((tower, index) => {
      tower.update(time, this.enemies)

      // 移除已銷毀的塔
      if (!tower.active) {
        this.towers.splice(index, 1)
      }
    })

    // 檢查是否完成當前波數
    if (this.enemiesSpawned >= this.enemiesInWave && this.enemies.length === 0) {
      this.nextWave()
    }
  }

  private nextWave() {
    this.waveNumber++
    this.enemiesSpawned = 0
    window.dispatchEvent(new CustomEvent('updateWave', { detail: this.waveNumber }))
    console.log(`開始第 ${this.waveNumber} 波，怪物血量增加！`)

    // 開始下一波
    this.spawnEnemies()
  }

  public setGameOver() {
    this.gameOver = true
  }
}

// UI 函數
const selectTower = (type: string) => {
  selectedTower.value = type
  window.dispatchEvent(new CustomEvent('selectTower', { detail: type }))
}

onMounted(() => {
  if (gameContainer.value) {
    game = new Phaser.Game({
      type: Phaser.AUTO,
      width: window.innerWidth,
      height: window.innerHeight,
      parent: gameContainer.value,
      backgroundColor: '#222',
      scene: GameScene,
      scale: {
        mode: Phaser.Scale.RESIZE,
        autoCenter: Phaser.Scale.CENTER_BOTH,
      },

    })

    // 監聽遊戲事件更新 UI
    window.addEventListener('updateMoney', ((event: CustomEvent) => {
      money.value = event.detail
    }) as EventListener)

    window.addEventListener('updateWave', ((event: CustomEvent) => {
      waveNumber.value = event.detail
    }) as EventListener)

    window.addEventListener('playerLoseHp', ((event: CustomEvent) => {
      playerHp.value -= event.detail
      if (playerHp.value <= 0 && !isGameOver.value) {
        isGameOver.value = true
        // 通知 Phaser 場景停止
        // eslint-disable-next-line @typescript-eslint/no-explicit-any
        const scene = game?.scene.getScene('GameScene') as any
        scene?.setGameOver()
      }
    }) as EventListener)
  }
})

onBeforeUnmount(() => {
  if (game) {
    game.destroy(true)
    game = null
  }
})
</script>

<style scoped>
.phaser-container {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  position: fixed;
  top: 0;
  left: 0;
  z-index: 0;
}

.tower-buttons {
  position: fixed;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 10px;
  z-index: 5;
}

.tower-buttons button {
  background: rgba(0, 0, 0, 0.8);
  color: white;
  border: 2px solid #666;
  padding: 12px 20px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 14px;
  font-weight: bold;
  transition: all 0.3s;
  min-width: 120px;
}

.tower-buttons button:hover {
  background: rgba(0, 0, 0, 0.9);
  border-color: #999;
  transform: translateY(-2px);
}

.tower-buttons button.selected {
  border-color: #00ff00;
  background: rgba(0, 255, 0, 0.3);
  box-shadow: 0 0 10px rgba(0, 255, 0, 0.5);
}

.ui-drawer {
  position: fixed;
  top: 0;
  right: 0;
  height: 100vh;
  z-index: 10;
  transition: transform 0.3s ease;
  transform: translateX(100%);
}

.ui-drawer.expanded {
  transform: translateX(0);
}

.drawer-handle {
  position: absolute;
  left: -50px;
  top: 50%;
  transform: translateY(-50%);
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 15px 10px;
  border-radius: 10px 0 0 10px;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 5px;
  transition: all 0.3s;
}

.drawer-handle:hover {
  background: rgba(0, 0, 0, 0.9);
}

.handle-icon {
  font-size: 20px;
  font-weight: bold;
}

.handle-text {
  font-size: 12px;
  writing-mode: vertical-rl;
  text-orientation: mixed;
}

.drawer-content {
  width: 300px;
  height: 100%;
  background: rgba(0, 0, 0, 0.9);
  color: white;
  padding: 20px;
  overflow-y: auto;
  font-family: Arial, sans-serif;
}

.game-info {
  margin-bottom: 20px;
}

.info-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  margin-bottom: 10px;
  border-radius: 5px;
  background: rgba(255, 255, 255, 0.1);
}

.info-item.hp {
  background: rgba(255, 0, 0, 0.3);
}

.info-item.money {
  background: rgba(255, 215, 0, 0.3);
}

.info-item.wave {
  background: rgba(0, 255, 0, 0.3);
}

.label {
  font-weight: bold;
}

.value {
  font-size: 18px;
  font-weight: bold;
}



.game-controls {
  margin-bottom: 20px;
}

.game-controls h3 {
  margin-bottom: 10px;
  color: #00ff00;
  font-size: 16px;
}

.game-controls button {
  width: 100%;
  background: #222;
  color: #fff;
  border: 2px solid #666;
  padding: 10px 15px;
  border-radius: 5px;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.2s;
  margin-bottom: 8px;
}

.game-controls button:hover {
  background: #444;
  border-color: #00ffcc;
}

.game-over {
  margin-top: 30px;
  font-size: 2.5rem;
  color: #ff3333;
  font-weight: bold;
  text-shadow: 2px 2px 8px #000;
  text-align: center;
}
</style>
