เยี่ยมเลยครับ! การที่จะทำให้เกมจำค่าได้แม้เราจะเปลี่ยนหน้าเว็บ (เปลี่ยน Tab/Route) ใน Vue.js แบบไม่ต้องใช้ระบบยุ่งยากอย่าง Vuex หรือ LocalStorage เราสามารถสร้าง **ตัวแปร Global ไว้ด้านนอกตัว Component** ได้เลยครับ 

นอกจากนี้ผมยังได้เพิ่ม **ระบบ Pause (พักเกม)** โดยสามารถกดได้ทั้งปุ่มบนจอ และกดปุ่ม **P** หรือ **ESC** บนคีย์บอร์ดครับ

คัดลอกโค้ดเวอร์ชันล่าสุดที่ "จำค่าได้ + พักเกมได้" นี้ไปทับใน `GameOnlyAdmin.vue` ได้เลยครับ:

```vue
<template>
  <div class="arcade-master">

    <div v-if="appState === 'MENU'" class="menu-container">
      <h1 class="main-title">ARCADE CENTER 🕹️</h1>
      <p class="subtitle">เลือกเกมที่คุณต้องการเล่น</p>
      
      <div class="game-select-grid">
        <div class="game-card snake-card" @click="launchGame('SNAKE')">
          <div class="card-icon">🐍</div>
          <h2>Snake Game</h2>
          <p>เกมงูคลาสสิก เลี้ยวซ้ายขวากินอาหาร</p>
        </div>
        <div class="game-card rogue-card" @click="launchGame('ROGUE')">
          <div class="card-icon">🛡️</div>
          <h2>Rogue Survivor</h2>
          <p>เอาชีวิตรอดจากซอมบี้ 8-Bit อัปเกรดตัวละคร</p>
        </div>
      </div>
    </div>

    <div v-if="appState === 'SNAKE'" class="game-area snake-area">
      <button class="top-btn back-btn" @click="backToMenu">⬅️ ออกไปหน้าเมนู</button>
      <button class="top-btn pause-btn" v-if="['PLAYING', 'PAUSED'].includes(s_gameState)" @click="s_togglePause">
        {{ s_gameState === 'PAUSED' ? '▶️ เล่นต่อ' : '⏸️ พักเกม' }}
      </button>

      <h1>เกมงู (Snake Game) 🐍</h1>
      <div class="snake-score-board">
        คะแนน: <strong>{{ s_score }}</strong> | สายพันธุ์: {{ s_snakeName }}
      </div>

      <div class="snake-board-wrapper">
        <div class="snake-game-board" :style="s_boardStyle">
          <div class="s-food" :style="s_getStyle(s_food)"></div>
          <div 
            v-for="(segment, index) in s_snake" :key="index"
            class="s-snake-segment" :class="[s_snakeType, { 's-snake-head': index === 0 }]"
            :style="s_getStyle(segment)"
          ></div>

          <div v-if="s_gameState === 'SELECT_SNAKE'" class="game-overlay">
            <h2>เลือกสายพันธุ์งู 🐍</h2>
            <div class="r-char-selection">
              <div class="r-char-card" @click="s_selectSnake('python', 220, 'งูหลาม')">
                <div class="sprite preview s-python"></div><h3>งูหลาม</h3><p>สปีด: ช้ามาก</p>
              </div>
              <div class="r-char-card" @click="s_selectSnake('green', 140, 'งูเขียว')">
                <div class="sprite preview s-green"></div><h3>งูเขียว</h3><p>สปีด: ปานกลาง</p>
              </div>
              <div class="r-char-card" @click="s_selectSnake('cobra', 80, 'งูเห่า')">
                <div class="sprite preview s-cobra"></div><h3>งูเห่า</h3><p>สปีด: เร็วมาก</p>
              </div>
            </div>
          </div>

          <div v-if="s_gameState === 'READY'" class="game-overlay">
            <h2>พร้อมไหม? 😎</h2><p>สายพันธุ์: {{ s_snakeName }}</p>
            <button class="action-btn start" @click="s_startGame">เริ่มเกมเลย!</button>
          </div>

          <div v-if="s_gameState === 'PAUSED'" class="game-overlay">
            <h2>⏸️ พักเกม</h2>
            <button class="action-btn start" @click="s_togglePause">เล่นต่อ</button>
          </div>

          <div v-if="s_gameState === 'GAMEOVER'" class="game-overlay">
            <h2>จบเกม! 💥</h2><p>คะแนนของคุณคือ: {{ s_score }}</p>
            <div style="display: flex; gap: 10px;">
              <button class="action-btn retry" @click="s_gameState = 'SELECT_SNAKE'">เปลี่ยนงู</button>
              <button class="action-btn start" @click="s_resetGame">เล่นงูเดิม</button>
            </div>
          </div>
        </div>
      </div>
      <p class="controls-hint">⌨️ กดลูกศร/WASD เพื่อเดิน | กด P หรือ ESC เพื่อพักเกม</p>
    </div>

    <div v-if="appState === 'ROGUE'" class="game-area rogue-area">
      <button class="top-btn back-btn" @click="backToMenu">⬅️ ออกไปหน้าเมนู</button>
      <button class="top-btn pause-btn" v-if="['PLAYING', 'PAUSED'].includes(r_gameState)" @click="r_togglePause">
        {{ r_gameState === 'PAUSED' ? '▶️ เล่นต่อ' : '⏸️ พักเกม' }}
      </button>

      <div class="r-ui-panel">
        <div class="r-stats-row">
          <span>LV: {{ r_player.level }} | EXP: {{ Math.floor(r_player.exp) }}/{{ Math.floor(r_player.nextExp) }}</span>
          <span>❤️ HP: {{ Math.ceil(r_player.hp) }}/{{ r_player.maxHp }}</span>
          <span class="r-gold-text">💰 {{ r_gold }}</span>
        </div>
        <div class="r-exp-bar-bg"><div class="r-exp-bar-fill" :style="{ width: r_expPercent + '%' }"></div></div>
        <div class="r-timer">Wave: {{ r_wave }} | Time: {{ r_timeLeft }}s ⏳</div>
      </div>

      <div class="r-game-world" ref="rogueWorld">
        <div class="sprite r-player" :class="[r_player.gender, { 'flip': r_player.facingLeft }]" :style="r_objectStyle(r_player)">
          <div class="r-hp-bar-mini"><div class="r-hp-fill-mini" :style="{ width: (r_player.hp/r_player.maxHp)*100 + '%' }"></div></div>
        </div>
        
        <div v-for="e in r_enemies" :key="'e'+e.id" class="sprite r-enemy" :class="{ 'boss': e.isBoss, 'flip': e.facingLeft }" :style="r_objectStyle(e)"></div>
        <div v-for="b in r_bullets" :key="'b'+b.id" class="r-bullet" :style="r_objectStyle(b)"></div>
        <div v-for="g in r_gems" :key="'g'+g.id" class="r-gem" :style="r_objectStyle(g)"></div>
        <div v-for="c in r_coins" :key="'c'+c.id" class="r-coin" :style="r_objectStyle(c)"></div>
        <div v-for="t in r_floatingTexts" :key="'t'+t.id" class="r-floating-text" :style="{ left: t.x + 'px', top: t.y + 'px', opacity: t.life / 30 }">{{ t.text }}</div>

        <div v-if="r_gameState === 'SELECT_CHAR'" class="game-overlay">
          <h2>เลือกฮีโร่ของคุณ</h2>
          <div class="r-char-selection">
            <div class="r-char-card" @click="r_selectChar('male')"><div class="sprite preview male"></div><h3>นักรบชาย</h3></div>
            <div class="r-char-card" @click="r_selectChar('female')"><div class="sprite preview female"></div><h3>นักรบหญิง</h3></div>
          </div>
        </div>

        <div v-if="r_gameState === 'PAUSED'" class="game-overlay">
          <h2>⏸️ พักเกม</h2>
          <button class="action-btn start" @click="r_togglePause">เล่นต่อ</button>
        </div>

        <div v-if="r_gameState === 'LEVEL_UP'" class="game-overlay">
          <h2>LEVEL UP! ✨ เลเวล {{ r_player.level }}</h2>
          <div class="r-upgrade-list">
            <button v-for="u in r_upgradeOptions" :key="u.name" class="r-upgrade-item" @click="r_applyUpgrade(u)">{{ u.label }}</button>
          </div>
        </div>

        <div v-if="r_gameState === 'SHOP'" class="game-overlay">
          <h2>ร้านค้าอาวุธ 🛒 (จบเวฟ {{ r_wave }})</h2>
          <p>คุณมีเงิน: <span class="r-gold-text">💰 {{ r_gold }}</span></p>
          <div class="r-shop-grid">
            <button v-for="(s, key) in r_shopItems" :key="key" class="r-shop-item" :disabled="r_gold < s.cost" @click="r_buyItem(key)">
              {{ s.label }}<br>💰{{ s.cost }}
            </button>
          </div>
          <button class="action-btn start" @click="r_startNextWave">ลุยเวฟ {{ r_wave + 1 }} ต่อ ⚔️</button>
        </div>

        <div v-if="r_gameState === 'GAMEOVER'" class="game-overlay">
          <h2>ภารกิจล้มเหลว! 💀</h2><p>เวฟที่ไปถึง: {{ r_wave }} | เลเวล: {{ r_player.level }}</p>
          <button class="action-btn retry" @click="r_gameState = 'SELECT_CHAR'">เลือกตัวละครใหม่</button>
        </div>
      </div>
      <p class="controls-hint">⌨️ WASD เดิน | กด P หรือ ESC เพื่อพักเกม | 🔫 โจมตีออโต้</p>
    </div>

  </div>
</template>

<script>
// ตัวแปร Global ไว้นอก Export Default เพื่อจดจำสถานะข้ามการเปลี่ยนหน้าจอ (Tab)
let persistenceState = null;

export default {
  name: 'GameOnlyAdmin',
  data() {
    const defaultData = {
      appState: 'MENU',

      // ========================
      // ข้อมูลเกมงู
      // ========================
      s_gameState: 'SELECT_SNAKE',
      s_snakeType: 'green', s_snakeName: 'ยังไม่เลือก', s_baseSpeed: 150,
      s_gridSize: 20, s_cellSize: 20, s_snake: [], s_food: { x: 0, y: 0 },
      s_dir: { x: 0, y: -1 }, s_nextDir: { x: 0, y: -1 }, s_score: 0, s_interval: null, s_speed: 150,

      // ========================
      // ข้อมูลเกม Rogue
      // ========================
      r_gameState: 'SELECT_CHAR',
      r_player: { x: 300, y: 250, r: 15, gender: 'male', speed: 3, hp: 100, maxHp: 100, level: 1, exp: 0, nextExp: 100, atkSpeed: 500, damage: 20, facingLeft: false },
      r_enemies: [], r_bullets: [], r_gems: [], r_coins: [], r_floatingTexts: [], r_keys: {},
      r_gold: 0, r_wave: 1, r_timeLeft: 30,
      r_enemyId: 0, r_bulletId: 0, r_gemId: 0, r_coinId: 0, r_textId: 0,
      r_gameLoop: null, r_spawnTimer: null, r_waveTimer: null,
      r_upgradeOptions: [],
      r_shopItems: {
        maxHp: { label: '❤️ Max HP +50', cost: 50, run: () => { this.r_player.maxHp += 50; this.r_player.hp += 50; } },
        damage: { label: '⚔️ พลังโจมตี +15', cost: 40, run: () => { this.r_player.damage += 15; } },
        atkSpeed: { label: '🔥 ยิงรัวขึ้น 10%', cost: 60, run: () => { this.r_player.atkSpeed *= 0.9; } },
        heal: { label: '🩹 ฟื้นฟูเลือดเต็ม', cost: 30, run: () => { this.r_player.hp = this.r_player.maxHp; } }
      }
    };

    // ถ้ามีสถานะเซฟไว้ตอนออกหน้าเว็บ ให้โหลดกลับมา
    if (persistenceState) {
      Object.assign(defaultData, persistenceState);
    }
    return defaultData;
  },
  computed: {
    s_boardStyle() { return { width: `${this.s_gridSize * this.s_cellSize}px`, height: `${this.s_gridSize * this.s_cellSize}px` }; },
    r_expPercent() { return Math.min((this.r_player.exp / this.r_player.nextExp) * 100, 100); }
  },
  created() {
    // กู้คืนฟังก์ชันการอัปเกรด (เนื่องจากฟังก์ชันเซฟเข้า Global ตรงๆ อาจมีปัญหา this)
    if (persistenceState && persistenceState.r_savedUpgrades) {
      const allUpgradesMap = {
        spd: { name: 'spd', label: '👟 วิ่งเร็วขึ้น', run: () => this.r_player.speed += 0.4 },
        dmg: { name: 'dmg', label: '⚔️ โจมตีแรงขึ้น', run: () => this.r_player.damage += 15 },
        atk: { name: 'atk', label: '🔥 ยิงรัวขึ้น', run: () => this.r_player.atkSpeed *= 0.85 },
        hp: { name: 'hp', label: '🛡️ Max HP +20', run: () => { this.r_player.maxHp+=20; this.r_player.hp+=20; } }
      };
      this.r_upgradeOptions = persistenceState.r_savedUpgrades.map(n => allUpgradesMap[n]);
    }
  },
  mounted() {
    window.addEventListener('keydown', this.handleKeydown);
    window.addEventListener('keyup', this.handleKeyup);
  },
  beforeDestroy() {
    // 1. ถอด Event Listener
    window.removeEventListener('keydown', this.handleKeydown);
    window.removeEventListener('keyup', this.handleKeyup);

    // 2. หยุดตัวจับเวลาทั้งหมด
    this.clearAllIntervals();

    // 3. ปรับสถานะเป็นพักเกมอัตโนมัติก่อนออก (กันกลับมาแล้วตายเลย)
    if (this.s_gameState === 'PLAYING') this.s_gameState = 'PAUSED';
    if (this.r_gameState === 'PLAYING') this.r_gameState = 'PAUSED';

    // 4. บันทึกข้อมูลทั้งหมดลง Global Variable
    persistenceState = {
      appState: this.appState,
      s_gameState: this.s_gameState, s_snakeType: this.s_snakeType, s_snakeName: this.s_snakeName, s_baseSpeed: this.s_baseSpeed,
      s_snake: this.s_snake, s_food: this.s_food, s_dir: this.s_dir, s_nextDir: this.s_nextDir, s_score: this.s_score, s_speed: this.s_speed,
      r_gameState: this.r_gameState, r_player: this.r_player, r_enemies: this.r_enemies, r_bullets: this.r_bullets,
      r_gems: this.r_gems, r_coins: this.r_coins, r_floatingTexts: this.r_floatingTexts,
      r_gold: this.r_gold, r_wave: this.r_wave, r_timeLeft: this.r_timeLeft,
      r_enemyId: this.r_enemyId, r_bulletId: this.r_bulletId, r_gemId: this.r_gemId, r_coinId: this.r_coinId, r_textId: this.r_textId,
      r_savedUpgrades: this.r_upgradeOptions.map(u => u.name) // จำแค่ชื่อ Option ไปประกอบร่างใหม่
    };
  },
  methods: {
    clearAllIntervals() {
      clearInterval(this.s_interval);
      clearInterval(this.r_gameLoop); clearInterval(this.r_spawnTimer); clearInterval(this.r_waveTimer);
    },
    backToMenu() {
      this.clearAllIntervals();
      persistenceState = null; // ล้างเซฟทิ้งเมื่อกดยอมแพ้ออกมาหน้าเมนู
      this.appState = 'MENU';
    },
    launchGame(gameName) {
      this.clearAllIntervals();
      this.appState = gameName;
      if (gameName === 'SNAKE') this.s_gameState = 'SELECT_SNAKE';
      else if (gameName === 'ROGUE') this.r_gameState = 'SELECT_CHAR';
    },
    handleKeydown(e) {
      if (['ArrowUp', 'ArrowDown', 'Space'].includes(e.code)) e.preventDefault();
      
      // ดักจับปุ่ม Pause
      if (['KeyP', 'Escape'].includes(e.code)) {
        if (this.appState === 'SNAKE' && ['PLAYING', 'PAUSED'].includes(this.s_gameState)) this.s_togglePause();
        else if (this.appState === 'ROGUE' && ['PLAYING', 'PAUSED'].includes(this.r_gameState)) this.r_togglePause();
        return;
      }

      if (this.appState === 'SNAKE') this.s_handleKeydown(e);
      else if (this.appState === 'ROGUE') this.$set(this.r_keys, e.code, true);
    },
    handleKeyup(e) { if (this.appState === 'ROGUE') this.$set(this.r_keys, e.code, false); },

    // ========================
    // ฟังก์ชันของเกม SNAKE
    // ========================
    s_togglePause() {
      if (this.s_gameState === 'PLAYING') {
        this.s_gameState = 'PAUSED';
        clearInterval(this.s_interval);
      } else if (this.s_gameState === 'PAUSED') {
        this.s_gameState = 'PLAYING';
        this.s_interval = setInterval(this.s_gameLoop, this.s_speed);
      }
    },
    s_getStyle(pos) { return { left: `${pos.x * this.s_cellSize}px`, top: `${pos.y * this.s_cellSize}px`, width: `${this.s_cellSize}px`, height: `${this.s_cellSize}px` }; },
    s_selectSnake(type, speed, name) { this.s_snakeType = type; this.s_baseSpeed = speed; this.s_snakeName = name; this.s_resetGame(); },
    s_setupBoard() {
      const mid = Math.floor(this.s_gridSize / 2);
      this.s_snake = [{ x: mid, y: mid }, { x: mid, y: mid + 1 }, { x: mid, y: mid + 2 }];
      this.s_dir = { x: 0, y: -1 }; this.s_nextDir = { x: 0, y: -1 };
      this.s_score = 0; this.s_speed = this.s_baseSpeed;
      this.s_generateFood();
    },
    s_resetGame() { this.s_setupBoard(); this.s_gameState = 'READY'; },
    s_startGame() {
      this.s_gameState = 'PLAYING';
      if (this.s_interval) clearInterval(this.s_interval);
      this.s_interval = setInterval(this.s_gameLoop, this.s_speed);
    },
    s_gameLoop() {
      if (this.s_gameState !== 'PLAYING') return;
      this.s_dir = { ...this.s_nextDir };
      const head = this.s_snake[0]; const newHead = { x: head.x + this.s_dir.x, y: head.y + this.s_dir.y };
      if (newHead.x < 0 || newHead.x >= this.s_gridSize || newHead.y < 0 || newHead.y >= this.s_gridSize) return this.s_gameOver();
      if (this.s_snake.some(s => s.x === newHead.x && s.y === newHead.y)) return this.s_gameOver();

      this.s_snake.unshift(newHead);
      if (newHead.x === this.s_food.x && newHead.y === this.s_food.y) {
        this.s_score += 10; this.s_generateFood();
        if (this.s_speed > 50) { this.s_speed -= 3; clearInterval(this.s_interval); this.s_interval = setInterval(this.s_gameLoop, this.s_speed); }
      } else { this.s_snake.pop(); }
    },
    s_generateFood() {
      let newFood; let isOccupied = true;
      while (isOccupied) { newFood = { x: Math.floor(Math.random() * this.s_gridSize), y: Math.floor(Math.random() * this.s_gridSize) }; isOccupied = this.s_snake.some(s => s.x === newFood.x && s.y === newFood.y); }
      this.s_food = newFood;
    },
    s_handleKeydown(e) {
      if (this.s_gameState === 'READY' && ['Enter', 'Space'].includes(e.code)) return this.s_startGame();
      if (['ArrowUp', 'KeyW'].includes(e.code) && this.s_dir.y === 0) this.s_nextDir = { x: 0, y: -1 };
      else if (['ArrowDown', 'KeyS'].includes(e.code) && this.s_dir.y === 0) this.s_nextDir = { x: 0, y: 1 };
      else if (['ArrowLeft', 'KeyA'].includes(e.code) && this.s_dir.x === 0) this.s_nextDir = { x: -1, y: 0 };
      else if (['ArrowRight', 'KeyD'].includes(e.code) && this.s_dir.x === 0) this.s_nextDir = { x: 1, y: 0 };
    },
    s_gameOver() { this.s_gameState = 'GAMEOVER'; clearInterval(this.s_interval); },

    // ========================
    // ฟังก์ชันของเกม ROGUE
    // ========================
    r_togglePause() {
      if (this.r_gameState === 'PLAYING') {
        this.r_gameState = 'PAUSED';
        clearInterval(this.r_gameLoop); clearInterval(this.r_spawnTimer); clearInterval(this.r_waveTimer);
      } else if (this.r_gameState === 'PAUSED') {
        this.r_gameState = 'PLAYING';
        this.r_gameLoop = setInterval(this.r_update, 1000 / 60);
        this.r_spawnTimer = setInterval(this.r_spawnEnemy, Math.max(500, 1500 - (this.r_wave * 100)));
        this.r_waveTimer = setInterval(() => { this.r_timeLeft--; if (this.r_timeLeft <= 0) this.r_finishWave(); }, 1000);
      }
    },
    r_objectStyle(obj) { const size = (obj.r || 5) * 2; return { left: `${obj.x - (obj.r || 5)}px`, top: `${obj.y - (obj.r || 5)}px`, width: `${size}px`, height: `${size}px` }; },
    r_selectChar(gender) {
      this.r_player.gender = gender;
      if (gender === 'male') { this.r_player.hp = 150; this.r_player.maxHp = 150; this.r_player.speed = 2.5; this.r_player.damage = 20; } 
      else { this.r_player.hp = 80; this.r_player.maxHp = 80; this.r_player.speed = 4; this.r_player.damage = 35; }
      this.r_gold = 0; this.r_wave = 1; this.r_player.level = 1; this.r_player.exp = 0; this.r_player.nextExp = 100; this.r_player.x = 300; this.r_player.y = 250;
      this.r_prepareWave();
    },
    r_startNextWave() { if (this.r_gameState === 'SHOP') this.r_wave++; this.r_prepareWave(); },
    r_prepareWave() {
      this.r_gameState = 'PLAYING'; this.r_timeLeft = 30 + (this.r_wave * 2);
      this.r_enemies = []; this.r_bullets = []; this.r_floatingTexts = [];
      clearInterval(this.r_gameLoop); clearInterval(this.r_spawnTimer); clearInterval(this.r_waveTimer);
      this.r_gameLoop = setInterval(this.r_update, 1000 / 60);
      this.r_spawnTimer = setInterval(this.r_spawnEnemy, Math.max(500, 1500 - (this.r_wave * 100)));
      this.r_waveTimer = setInterval(() => { this.r_timeLeft--; if (this.r_timeLeft <= 0) this.r_finishWave(); }, 1000);
    },
    r_spawnEnemy() {
      if (this.r_gameState !== 'PLAYING') return;
      const side = Math.floor(Math.random() * 4); let x, y;
      if (side === 0) { x = Math.random() * 600; y = -20; } else if (side === 1) { x = 620; y = Math.random() * 500; }
      else if (side === 2) { x = Math.random() * 600; y = 520; } else { x = -20; y = Math.random() * 500; }
      if ((this.r_wave % 5 === 0) && (Math.random() < 0.1)) this.r_enemies.push({ id: this.r_enemyId++, x, y, r: 25, hp: 200 + (this.r_wave * 50), speed: 0.8, isBoss: true, facingLeft: false });
      else this.r_enemies.push({ id: this.r_enemyId++, x, y, r: 12, hp: 30 + (this.r_wave * 15), speed: 1.2 + (this.r_wave * 0.05), isBoss: false, facingLeft: false });
    },
    r_update() {
      if (this.r_gameState !== 'PLAYING') return;
      let dx = 0, dy = 0;
      if ((this.r_keys['KeyW'] || this.r_keys['ArrowUp']) && this.r_player.y > 0) dy -= this.r_player.speed;
      if ((this.r_keys['KeyS'] || this.r_keys['ArrowDown']) && this.r_player.y < 500) dy += this.r_player.speed;
      if ((this.r_keys['KeyA'] || this.r_keys['ArrowLeft']) && this.r_player.x > 0) { dx -= this.r_player.speed; this.r_player.facingLeft = true; }
      if ((this.r_keys['KeyD'] || this.r_keys['ArrowRight']) && this.r_player.x < 600) { dx += this.r_player.speed; this.r_player.facingLeft = false; }
      this.r_player.x += dx; this.r_player.y += dy;

      const target = this.r_enemies[0] || null;
      if (target && Date.now() - (this.r_lastShot || 0) > this.r_player.atkSpeed) {
        const angle = Math.atan2(target.y - this.r_player.y, target.x - this.r_player.x);
        this.r_bullets.push({ id: this.r_bulletId++, x: this.r_player.x, y: this.r_player.y, vx: Math.cos(angle) * 10, vy: Math.sin(angle) * 10, r: 6 }); this.r_lastShot = Date.now();
      }
      this.r_bullets.forEach((b, i) => { b.x += b.vx; b.y += b.vy; if (b.x < -20 || b.x > 620 || b.y < -20 || b.y > 520) this.r_bullets.splice(i, 1); });

      this.r_enemies.forEach((e, ei) => {
        const angle = Math.atan2(this.r_player.y - e.y, this.r_player.x - e.x);
        e.x += Math.cos(angle) * e.speed; e.y += Math.sin(angle) * e.speed; e.facingLeft = this.r_player.x < e.x;
        if (this.r_dist(e, this.r_player) < e.r + this.r_player.r - 5) { this.r_player.hp -= e.isBoss ? 1.5 : 0.4; if (this.r_player.hp <= 0) this.r_gameOver(); }

        this.r_bullets.forEach((b, bi) => {
          if (this.r_dist(e, b) < e.r + b.r) {
            e.hp -= this.r_player.damage; this.r_bullets.splice(bi, 1);
            this.r_floatingTexts.push({ id: this.r_textId++, x: e.x, y: e.y - 10, text: this.r_player.damage, life: 30 });
            if (e.hp <= 0) {
              const dropCount = e.isBoss ? 5 : 1;
              for(let i=0; i<dropCount; i++) {
                const ox = (Math.random() - 0.5) * 20; const oy = (Math.random() - 0.5) * 20;
                if (Math.random() > 0.4) this.r_gems.push({ id: this.r_gemId++, x: e.x + ox, y: e.y + oy, r: 6 });
                else this.r_coins.push({ id: this.r_coinId++, x: e.x + ox, y: e.y + oy, r: 6 });
              }
              this.r_enemies.splice(ei, 1);
            }
          }
        });
      });

      this.r_gems.forEach((g, gi) => { if (this.r_dist(g, this.r_player) < 60) { g.x += (this.r_player.x - g.x) * 0.15; g.y += (this.r_player.y - g.y) * 0.15; if (this.r_dist(g, this.r_player) < 20) { this.r_gems.splice(gi, 1); this.r_gainExp(30 + this.r_wave*5); } } });
      this.r_coins.forEach((c, ci) => { if (this.r_dist(c, this.r_player) < 60) { c.x += (this.r_player.x - c.x) * 0.15; c.y += (this.r_player.y - c.y) * 0.15; if (this.r_dist(c, this.r_player) < 20) { this.r_coins.splice(ci, 1); this.r_gold += (Math.floor(Math.random()*3) + 2); } } });
      this.r_floatingTexts.forEach((t, i) => { t.y -= 1; t.life -= 1; if (t.life <= 0) this.r_floatingTexts.splice(i, 1); });
    },
    r_finishWave() { this.r_gameState = 'SHOP'; clearInterval(this.r_gameLoop); clearInterval(this.r_spawnTimer); clearInterval(this.r_waveTimer); this.r_enemies = []; },
    r_buyItem(key) { const item = this.r_shopItems[key]; if (this.r_gold >= item.cost) { this.r_gold -= item.cost; item.run(); item.cost = Math.floor(item.cost * 1.3); } },
    r_gainExp(val) {
      this.r_player.exp += val;
      if (this.r_player.exp >= this.r_player.nextExp) {
        this.r_player.level++; this.r_player.exp -= this.r_player.nextExp; this.r_player.nextExp = Math.floor(this.r_player.nextExp * 1.3);
        this.r_gameState = 'LEVEL_UP';
        const allUpgradesMap = {
          spd: { name: 'spd', label: '👟 วิ่งเร็วขึ้น', run: () => this.r_player.speed += 0.4 }, dmg: { name: 'dmg', label: '⚔️ โจมตีแรงขึ้น', run: () => this.r_player.damage += 15 },
          atk: { name: 'atk', label: '🔥 ยิงรัวขึ้น', run: () => this.r_player.atkSpeed *= 0.85 }, hp: { name: 'hp', label: '🛡️ Max HP +20', run: () => { this.r_player.maxHp+=20; this.r_player.hp+=20; } }
        };
        this.r_upgradeOptions = ['spd', 'dmg', 'atk', 'hp'].sort(() => 0.5 - Math.random()).slice(0, 3).map(k => allUpgradesMap[k]);
      }
    },
    r_applyUpgrade(u) { u.run(); this.r_gameState = 'PLAYING'; },
    r_gameOver() { this.r_gameState = 'GAMEOVER'; clearInterval(this.r_gameLoop); clearInterval(this.r_spawnTimer); clearInterval(this.r_waveTimer); },
    r_dist(a, b) { return Math.hypot(a.x - b.x, a.y - b.y); }
  }
};
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Press+Start+2P&family=Sarabun:wght@400;700&display=swap');

.arcade-master { display: flex; flex-direction: column; align-items: center; justify-content: center; min-height: 100vh; background-color: #0d1117; color: #fff; font-family: 'Sarabun', sans-serif; user-select: none; }
h1, h2 { font-family: 'Press Start 2P', 'Sarabun', cursive; margin-bottom: 10px; }

/* ปุ่มด้านบน */
.top-btn { position: absolute; top: 20px; padding: 10px 15px; color: white; border: none; border-radius: 8px; cursor: pointer; font-weight: bold; font-family: 'Sarabun', sans-serif; z-index: 200; box-shadow: 0 4px 6px rgba(0,0,0,0.3); transition: 0.1s;}
.top-btn:active { transform: translateY(2px); }
.back-btn { left: 20px; background: #34495e; }
.back-btn:hover { background: #e74c3c; }
.pause-btn { right: 20px; background: #f39c12; }
.pause-btn:hover { background: #e67e22; }

.menu-container { text-align: center; }
.main-title { font-size: 2.5rem; color: #f1c40f; text-shadow: 4px 4px 0 #d35400; margin-bottom: 5px; }
.subtitle { font-size: 1.2rem; color: #aaa; margin-bottom: 40px; }
.game-select-grid { display: flex; gap: 30px; justify-content: center; }
.game-card { background: #1f2937; padding: 30px; border-radius: 16px; width: 260px; cursor: pointer; border: 4px solid #374151; transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1); box-shadow: 0 10px 20px rgba(0,0,0,0.5); }
.game-card:hover { transform: translateY(-10px); border-color: #3498db; box-shadow: 0 15px 30px rgba(52,152,219,0.4); }
.game-card.rogue-card:hover { border-color: #e74c3c; box-shadow: 0 15px 30px rgba(231,76,60,0.4); }
.card-icon { font-size: 4rem; margin-bottom: 15px; }
.game-card h2 { font-size: 1.2rem; margin-bottom: 10px; }
.game-card p { font-size: 0.9rem; color: #9ca3af; }

.game-area { display: flex; flex-direction: column; align-items: center; position: relative; width: 100%; height: 100vh; padding-top: 30px;}
.game-overlay { position: absolute; inset: 0; background: rgba(0,0,0,0.85); display: flex; flex-direction: column; justify-content: center; align-items: center; z-index: 100; text-align: center; backdrop-filter: blur(3px); }
.action-btn { padding: 12px 25px; background: #2ecc71; border: 3px solid #27ae60; color: white; border-radius: 8px; cursor: pointer; font-weight: bold; font-size: 1.1rem; font-family: 'Sarabun', sans-serif; transition: 0.1s; margin-top: 15px; }
.action-btn:hover { transform: scale(1.05); }
.action-btn.retry { background: #3498db; border-color: #2980b9; }
.controls-hint { margin-top: 15px; color: #888; font-size: 0.9rem; }

.snake-score-board { margin-bottom: 20px; font-size: 1.3rem; background: #2c3e50; padding: 10px 25px; border-radius: 12px; border: 2px solid #34495e;}
.snake-board-wrapper { padding: 10px; background-color: #34495e; border-radius: 8px; box-shadow: 0 8px 16px rgba(0,0,0,0.5); }
.snake-game-board { position: relative; background-color: #1a252f; overflow: hidden; background-image: linear-gradient(#2c3e50 1px, transparent 1px), linear-gradient(90deg, #2c3e50 1px, transparent 1px); background-size: 20px 20px; }
.s-snake-segment { position: absolute; border-radius: 4px; box-sizing: border-box; }
.s-snake-segment.green { background-color: #2ecc71; border: 1px solid #27ae60; }
.s-snake-segment.green.s-snake-head { background-color: #f1c40f; }
.s-snake-segment.python { background-color: #e67e22; border: 1px solid #d35400; }
.s-snake-segment.python.s-snake-head { background-color: #f39c12; }
.s-snake-segment.cobra { background-color: #8e44ad; border: 1px solid #2c3e50; }
.s-snake-segment.cobra.s-snake-head { background-color: #e74c3c; }
.s-food { position: absolute; background-color: #e74c3c; border-radius: 50%; transform: scale(0.8); box-shadow: 0 0 8px #e74c3c; }
.s-python { background-color: #e67e22; border-radius: 50%; } .s-green { background-color: #2ecc71; border-radius: 50%; } .s-cobra { background-color: #8e44ad; border-radius: 50%; }

.r-ui-panel { width: 600px; padding: 15px; background: #1f2937; border-bottom: 4px solid #374151; border-radius: 8px 8px 0 0; }
.r-stats-row { display: flex; justify-content: space-between; font-weight: bold; font-size: 1.1rem; margin-bottom: 8px; }
.r-exp-bar-bg { width: 100%; height: 12px; background: #111827; border: 2px solid #4b5563; border-radius: 6px; overflow: hidden; }
.r-exp-bar-fill { height: 100%; background: linear-gradient(90deg, #00d2ff, #3a7bd5); transition: width 0.3s; }
.r-gold-text { color: #f1c40f; text-shadow: 1px 1px 0 #000; }
.r-timer { margin-top: 8px; font-weight: bold; text-align: center; color: #ff9f43; font-size: 1.2rem; }
.r-game-world { width: 600px; height: 500px; background-color: #202b20; position: relative; overflow: hidden; border: 4px solid #374151; border-radius: 0 0 8px 8px; box-shadow: inset 0 0 50px rgba(0,0,0,0.5); }
.r-game-world::before { content: ""; position: absolute; inset: 0; opacity: 0.1; background-image: radial-gradient(#000 1px, transparent 1px); background-size: 20px 20px; }

.sprite { position: absolute; background-size: cover; image-rendering: pixelated; transition: transform 0.1s; }
.sprite.flip { transform: scaleX(-1); }
.male { background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 8 8'%3E%3Crect x='3' y='0' width='2' height='2' fill='%23f1c27d'/%3E%3Crect x='2' y='2' width='4' height='3' fill='%233498db'/%3E%3Crect x='2' y='5' width='1' height='3' fill='%232c3e50'/%3E%3Crect x='5' y='5' width='1' height='3' fill='%232c3e50'/%3E%3C/svg%3E"); }
.female { background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 8 8'%3E%3Crect x='3' y='1' width='2' height='2' fill='%23f1c27d'/%3E%3Crect x='2' y='0' width='4' height='1' fill='%23e74c3c'/%3E%3Crect x='1' y='1' width='1' height='3' fill='%23e74c3c'/%3E%3Crect x='6' y='1' width='1' height='3' fill='%23e74c3c'/%3E%3Crect x='3' y='3' width='2' height='2' fill='%239b59b6'/%3E%3Crect x='2' y='5' width='4' height='1' fill='%238e44ad'/%3E%3Crect x='3' y='6' width='2' height='2' fill='%232c3e50'/%3E%3C/svg%3E"); }
.r-enemy { background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 8 8'%3E%3Crect x='2' y='1' width='4' height='3' fill='%232ecc71'/%3E%3Crect x='3' y='2' width='1' height='1' fill='%23000'/%3E%3Crect x='5' y='2' width='1' height='1' fill='%23000'/%3E%3Crect x='2' y='4' width='4' height='2' fill='%238e44ad'/%3E%3Crect x='1' y='4' width='1' height='1' fill='%232ecc71'/%3E%3Crect x='6' y='4' width='1' height='1' fill='%232ecc71'/%3E%3Crect x='2' y='6' width='1' height='2' fill='%2334495e'/%3E%3Crect x='5' y='6' width='1' height='2' fill='%2334495e'/%3E%3C/svg%3E"); }
.r-enemy.boss { background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 8 8'%3E%3Crect x='1' y='1' width='6' height='4' fill='%23c0392b'/%3E%3Crect x='2' y='2' width='1' height='1' fill='%23f1c40f'/%3E%3Crect x='5' y='2' width='1' height='1' fill='%23f1c40f'/%3E%3Crect x='1' y='5' width='6' height='3' fill='%232c3e50'/%3E%3C/svg%3E"); filter: drop-shadow(0 0 10px red); }
.r-player { z-index: 10; filter: drop-shadow(0 4px 4px rgba(0,0,0,0.5)); }
.r-hp-bar-mini { position: absolute; top: -8px; left: -10%; width: 120%; height: 5px; background: #c0392b; border: 1px solid #000; border-radius: 2px; }
.r-hp-fill-mini { height: 100%; background: #2ecc71; }
.r-bullet { position: absolute; background: #fff; border: 2px solid #f1c40f; border-radius: 50%; box-shadow: 0 0 8px #f1c40f; }
.r-gem { position: absolute; background: #9b59b6; border: 1px solid #fff; transform: rotate(45deg); box-shadow: 0 0 5px #9b59b6; }
.r-coin { position: absolute; background: #f1c40f; border-radius: 50%; border: 2px dashed #d35400; animation: spin 2s linear infinite; }
@keyframes spin { 100% { transform: rotate(360deg); } }
.r-floating-text { position: absolute; color: #ff6b6b; font-weight: bold; font-size: 1.2rem; pointer-events: none; text-shadow: 1px 1px 0 #000, -1px -1px 0 #000; z-index: 20; }

.r-char-selection { display: flex; gap: 20px; margin-top: 20px; }
.r-char-card { background: #333; padding: 20px; border-radius: 12px; cursor: pointer; border: 3px solid #555; width: 150px; text-align: center;}
.r-char-card p { font-size: 0.8rem; color: #aaa; margin-top: 8px;}
.r-char-card:hover { border-color: #00d2ff; background: #444; }
.preview { position: relative; width: 64px; height: 64px; margin: 0 auto 10px; }

.r-shop-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 20px 0; width: 80%; }
.r-shop-item { padding: 15px; background: #2c3e50; color: white; border: 2px solid #34495e; border-radius: 8px; cursor: pointer; font-family: 'Sarabun', sans-serif; }
.r-shop-item:hover:not(:disabled) { border-color: #f1c40f; transform: scale(1.05); }
.r-shop-item:disabled { opacity: 0.4; cursor: not-allowed; }

.r-upgrade-list { display: flex; flex-direction: column; gap: 12px; width: 300px; margin-top: 20px; }
.r-upgrade-item { padding: 15px; background: linear-gradient(135deg, #2980b9, #8e44ad); color: white; border: 2px solid #fff; border-radius: 8px; cursor: pointer; font-size: 1.1rem; font-weight: bold; }
.r-upgrade-item:hover { transform: scale(1.05); }
</style>
```

**สิ่งที่เพิ่มเข้ามา:**
1.  **เซฟสถานะเกมข้ามหน้า (Persistence):** เมื่อคุณกดเปลี่ยน Tab ไปเมนูอื่นใน Vue.js โค้ดจะดักจับเหตุการณ์ `beforeDestroy` (ก่อนหน้าที่ Component จะถูกทำลาย) เพื่อเซฟค่าและพิกัดทุกอย่างลงใน `persistenceState` และเมื่อคุณคลิกกลับมาที่ `GameOnlyAdmin.vue` โค้ดจะโหลดทุกอย่างกลับมาจากเซฟครับ
2.  **พักเกมอัตโนมัติ (Auto-Pause):** ตอนสลับหน้าเว็บไปที่อื่น เกมจะสลับเข้าสู่โหมด "PAUSED" ให้เองอัตโนมัติ พอกลับมาคุณก็แค่กดปุ่ม "เล่นต่อ" (กันปัญหาจู่ๆ กลับมาแล้วซอมบี้รุมกัดตาย)
3.  **ปุ่มควบคุม (Pause/Back):** เพิ่มปุ่มสำหรับ "ออกไปหน้าเมนู" ไว้มุมซ้ายบน และปุ่ม "พักเกม" ไว้มุมขวาบน ใช้งานง่ายขึ้นครับ