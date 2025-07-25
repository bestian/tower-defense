<template>
  <div ref="gameContainer" class="phaser-container"></div>
  <div class="ui-overlay">
    <div class="hp">玩家血量: {{ playerHp }}</div>
    <div class="money">金錢: ${{ money }}</div>
    <div class="wave">波數: {{ waveNumber }}</div>
    <div class="tower-menu">
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
    <div class="game-controls">
      <button @click="togglePause">{{ isPaused ? '繼續' : '暫停' }}</button>
      <button @click="restartGame">重新開始</button>
    </div>
    <div v-if="isGameOver" class="game-over">GAME OVER</div>
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

function restartGame() {
  isGameOver.value = false
  isPaused.value = false
  playerHp.value = 100
  money.value = 1000
  waveNumber.value = 1
  selectedTower.value = null
  // 重新啟動 Phaser 場景
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  const scene = game?.scene.getScene('GameScene') as any
  scene?.scene.restart()
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
class Tower extends Phaser.GameObjects.Graphics {
  protected range: number = 0
  protected damage: number = 0
  protected attackSpeed: number = 0
  protected lastAttackTime: number = 0
  protected enemies: Enemy[] = []
  protected towerType: string
  private flyingAttackTimer: number = 0; // 新增飛行塔攻擊計時器
  private rangeCircle: Phaser.GameObjects.Graphics | null = null

  constructor(scene: Phaser.Scene, x: number, y: number, towerType: string) {
    super(scene)

    this.towerType = towerType
    this.enemies = []

    // 根據塔類型設定屬性
    switch (towerType) {
      case 'melee':
        this.range = 80
        this.damage = 60
        this.attackSpeed = 1000 // 1秒攻擊一次
        this.drawMeleeTower()
        break
      case 'ranged':
        this.range = 150
        this.damage = 20
        this.attackSpeed = 800 // 0.8秒攻擊一次
        this.drawRangedTower()
        break
      case 'flying':
        this.range = 100
        this.damage = 5
        this.attackSpeed = 1200 // 1.2秒攻擊一次
        this.drawFlyingTower()
        break
    }

    this.setPosition(x, y)
    scene.add.existing(this)
    // 顯示射程圈（近程與遠程塔）
    if (towerType === 'melee' || towerType === 'ranged') {
      this.rangeCircle = scene.add.graphics()
      this.drawRangeCircle()
    }
  }

  private drawMeleeTower() {
    // 近程塔：藍色方形
    this.fillStyle(0x0066ff)
    this.fillRect(-20, -20, 40, 40)
    this.lineStyle(2, 0xffffff)
    this.strokeRect(-20, -20, 40, 40)
  }

  private drawRangedTower() {
    // 遠程塔：紫色圓形
    this.fillStyle(0x9900ff)
    this.fillCircle(0, 0, 25)
    this.lineStyle(2, 0xffffff)
    this.strokeCircle(0, 0, 25)
  }

  private drawFlyingTower() {
    // 飛行塔：橙色三角形
    this.fillStyle(0xff6600)
    this.fillTriangle(-20, 20, 20, 20, 0, -20)
    this.lineStyle(2, 0xffffff)
    this.strokeTriangle(-20, 20, 20, 20, 0, -20)
  }

  private drawRangeCircle() {
    if (!this.rangeCircle) return
    this.rangeCircle.clear()
    this.rangeCircle.fillStyle(0x3399ff, 0.18)
    this.rangeCircle.lineStyle(1, 0x3399ff, 0.4)
    this.rangeCircle.fillCircle(this.x, this.y, this.range)
    this.rangeCircle.strokeCircle(this.x, this.y, this.range)
  }

  update(time: number, enemies: Enemy[]) {
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
    super.destroy(fromScene)
  }
}

// 怪物類別
class Enemy extends Phaser.GameObjects.Graphics {
  private path: Phaser.Curves.Path
  private pathProgress: number = 0
  private speed: number
  private health: number
  private maxHealth: number
  private healthBar: Phaser.GameObjects.Graphics | null = null

  constructor(scene: Phaser.Scene, path: Phaser.Curves.Path, health: number = 100) {
    super(scene)

    this.path = path
    this.maxHealth = health
    this.health = health
    this.speed = 0.00005 // 調慢敵人速度

    // 畫怪物（紅色圓形）
    this.fillStyle(0xff0000)
    this.fillCircle(0, 0, 15)
    this.lineStyle(2, 0xffffff)
    this.strokeCircle(0, 0, 15)

    scene.add.existing(this)

    // 設定初始位置在路徑起點
    const startPoint = this.path.getStartPoint()
    this.setPosition(startPoint.x, startPoint.y)

    // 血條
    this.healthBar = scene.add.graphics()
    this.updateHealthBar()
  }

  update(time: number, delta: number) {
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
  }

  takeDamage(damage: number) {
    this.health -= damage
    if (this.health <= 0) {
      // 怪物死亡，給予金錢獎勵
      const reward = 20 + Math.floor(this.maxHealth / 10)
      window.dispatchEvent(new CustomEvent('enemyKilled', { detail: reward }))
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
    // 響應式建立 S 型路徑
    this.createResponsiveSPath()

    // 畫出路徑（可視化用，之後可以隱藏）
    this.drawPath()

    // 開始生成怪物
    this.spawnEnemies()

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
    if (this.enemiesSpawned < this.enemiesInWave) {
      // 計算怪物血量（隨波數增加）
      const baseHealth = 100
      const healthIncrease = (this.waveNumber - 1) * 20
      const enemyHealth = baseHealth + healthIncrease

      const enemy = new Enemy(this, this.path, enemyHealth)
      this.enemies.push(enemy)
      this.enemiesSpawned++

      // 每 1 秒生成一隻怪物
      this.time.delayedCall(1000, () => {
        this.spawnEnemies()
      })
    }
  }

  private buildTower(x: number, y: number, type: string) {
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
      enemy.update(time, delta)

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

.ui-overlay {
  position: fixed;
  top: 20px;
  left: 20px;
  z-index: 10;
  color: white;
  font-family: Arial, sans-serif;
  font-size: 18px;
}

.money, .wave {
  margin-bottom: 10px;
  background: rgba(0, 0, 0, 0.7);
  padding: 10px;
  border-radius: 5px;
}

.tower-menu {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.tower-menu button {
  background: rgba(0, 0, 0, 0.7);
  color: white;
  border: 2px solid #666;
  padding: 10px 15px;
  border-radius: 5px;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.3s;
}

.tower-menu button:hover {
  background: rgba(0, 0, 0, 0.9);
  border-color: #999;
}

.tower-menu button.selected {
  border-color: #00ff00;
  background: rgba(0, 255, 0, 0.2);
}

.hp {
  margin-bottom: 10px;
  background: rgba(255, 0, 0, 0.7);
  padding: 10px;
  border-radius: 5px;
  font-weight: bold;
}
.game-over {
  margin-top: 30px;
  font-size: 2.5rem;
  color: #ff3333;
  font-weight: bold;
  text-shadow: 2px 2px 8px #000;
}
.game-controls {
  margin-top: 15px;
  display: flex;
  gap: 10px;
}
.game-controls button {
  background: #222;
  color: #fff;
  border: 2px solid #666;
  padding: 8px 18px;
  border-radius: 5px;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.2s;
}
.game-controls button:hover {
  background: #444;
  border-color: #00ffcc;
}
</style>
