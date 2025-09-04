<script setup>
import { ref, reactive, computed, onMounted, onUnmounted } from 'vue';
import OmegaNum from './OmegaNum.js';

// 游戏状态
const currentTab = ref('game');
// 圣魂系统状态迁移到player对象
const saveData = ref(null);
const saveDataText = ref('');
const isFighting = ref(true);
const isPaused = ref(true);
const isContinuous = ref(true);
const player = reactive({
  level: new OmegaNum(1),
  souls: new OmegaNum(0),
  maxDefeatedLevel: new OmegaNum(1),
  sacredSouls: new OmegaNum(0),
  upgrades: {
    1: new OmegaNum(1),
    2: new OmegaNum(1),
    3: new OmegaNum(1),
    4: new OmegaNum(1),
  },
});
const enemyMaxLife = ref(new OmegaNum(2));
const enemyCurrentLife = ref(new OmegaNum(2));

const enemyLevel = ref(new OmegaNum(1));
const fightInterval = ref(null);

// 计算属性
const totalDamage=computed(()=> player.level.mul(player.upgrades[1].gt(0) ? player.upgrades[1] : new OmegaNum(1)))
const totalSoulGain=computed(()=> enemyLevel.value.mul(player.upgrades[2].gt(0) ? player.upgrades[2] : new OmegaNum(1)))
const totalEnemyLife=computed(()=>
OmegaNum.pow(2, enemyLevel.value).div(player.upgrades[3].gt(0) ? player.upgrades[3] : new OmegaNum(1)));
const totalLevelCost=computed(()=>
OmegaNum.pow(2, player.level).div(player.upgrades[4].gt(0) ? player.upgrades[4] : new OmegaNum(1)));
const canRefine = computed(() => 
player.level.gte(10) && player.maxDefeatedLevel.gte(10));
const maxSelectableEnemyLevel = computed(() => 
  new OmegaNum(player.maxDefeatedLevel).toNumber());
const lifeBarWidth = computed(() => {
  const current = new OmegaNum(enemyCurrentLife.value || 0).toNumber();
  const max = new OmegaNum(enemyMaxLife.value || 1).toNumber();
  return (current / max * 100) + '%';
});

// 炼魂功能
function refineSouls() {
  player.sacredSouls = player.sacredSouls.add(
    player.level.mul(player.maxDefeatedLevel).pow(0.5)
  );
  player.level = new OmegaNum(1);
  player.maxDefeatedLevel = new OmegaNum(1);
  player.souls = new OmegaNum(0);
  enemyLevel.value = new OmegaNum(1);
  enemyMaxLife.value = totalEnemyLife.value;
  enemyCurrentLife.value = new OmegaNum(enemyMaxLife.value);
}

// 圣魂升级功能
function upgradeSacred(type) {
  let cost = OmegaNum.pow(2, player.upgrades[type]);
  if (player.sacredSouls.gte(cost)) {
    player.sacredSouls = player.sacredSouls.sub(cost);
    player.upgrades[type] = player.upgrades[type].add(1);
  }
}

// 升级功能
function levelUp() {
  if (player.souls.gte(totalLevelCost.value)) {
    player.souls = player.souls.sub(totalLevelCost.value);
    player.level = player.level.add(1);
  }
}

// 选择敌人等级
function selectEnemyLevel(level) {
  enemyLevel.value = new OmegaNum(level);
  enemyMaxLife.value = totalEnemyLife.value;
  enemyCurrentLife.value = new OmegaNum(enemyMaxLife.value);
}

// 开始战斗循环
function startFightLoop() {
  enemyMaxLife.value = totalEnemyLife.value;
  enemyCurrentLife.value = new OmegaNum(enemyMaxLife.value);
  
  return setInterval(() => {
    if (isPaused.value) return;
    enemyCurrentLife.value = enemyCurrentLife.value.sub(totalDamage.value.div(10));
    if (enemyCurrentLife.value.lte(0)) {
      endFight(true);
    }
  }, 100);
}

// 切换战斗状态
function toggleFight() {
  isPaused.value = !isPaused.value;
  
  if (!fightInterval.value && !isPaused.value) {
    fightInterval.value = startFightLoop();
  }
}

// 结束战斗
function endFight(isVictory) {
  clearInterval(fightInterval.value);
  fightInterval.value = null;
  isPaused.value = true;
  
  if (isVictory) {
      // 获得灵魂奖励
      player.souls = player.souls.add(totalSoulGain.value);
      
      // 更新最高击败等级
      if (enemyLevel.value.add(1).gte(player.maxDefeatedLevel)) {
        player.maxDefeatedLevel = enemyLevel.value.add(1);
      }
      
      // 重置敌人生命
      enemyMaxLife.value = totalEnemyLife.value;
      enemyCurrentLife.value = enemyMaxLife.value.clone();
      
      // 持续战斗逻辑
      if (isContinuous.value) {
        setTimeout(() => {
          isPaused.value = false;
          if (!fightInterval.value) {
            // 重置敌人生命值并直接启动战斗
            enemyCurrentLife.value = new OmegaNum(enemyMaxLife.value);
            fightInterval.value = setInterval(() => {
              if (isPaused.value) return;
              enemyCurrentLife.value = new OmegaNum(enemyCurrentLife.value).sub(totalDamage.value.div(10));
              if (enemyCurrentLife.value.lte(new OmegaNum(0))) {
                endFight(true);
              }
            }, 100);
          }
        }, 1000);
      }
    }
}

// 自动保存相关变量
const autoSaveInterval = ref(null);
const lastAutoSaveTime = ref(null);

// 存档功能
function saveGame() {
  const data = {
    player: {
      level: player.level.toString(),
      souls: player.souls.toString(),
      maxDefeatedLevel: player.maxDefeatedLevel.toString(),
      sacredSouls: player.sacredSouls.toString(),
      upgrades: {
        1: player.upgrades[1].toString(),
        2: player.upgrades[2].toString(),
        3: player.upgrades[3].toString(),
        4: player.upgrades[4].toString()
      }
    }
  };
  localStorage.setItem('gameSave', JSON.stringify(data));
  saveData.value = data;
}

function exportAsText(){
    saveDataText.value = JSON.stringify(saveData.value || {
        player: {
          level: player.level,
          souls: player.souls,
          maxDefeatedLevel: player.maxDefeatedLevel,
          sacredSouls: player.sacredSouls,
          upgrades: {
            1: player.upgrades[1],
            2: player.upgrades[2],
            3: player.upgrades[3],
            4: player.upgrades[4]
          }
        }
      });
}

function exportSave() {
  const data = JSON.stringify(saveData.value || {
    player: {
          level: player.level,
          souls: player.souls,
          maxDefeatedLevel: player.maxDefeatedLevel,
          sacredSouls: player.sacredSouls,
          upgrades: {
            1: player.upgrades[1],
            2: player.upgrades[2],
            3: player.upgrades[3],
            4: player.upgrades[4]
          }
        }
  });

  const blob = new Blob([data], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `qtrnity-save-${new Date().toISOString().slice(0,10)}.json`;
  a.click();
  URL.revokeObjectURL(url);
}

function importFromText() {
  try {
    const importedData = JSON.parse(saveDataText.value);
    // 验证存档数据结构
    if ((importedData.player?.level || importedData.playerLevel) && (importedData.player?.souls || importedData.playerSouls)) {
      saveData.value = importedData;
      localStorage.setItem('gameSave', JSON.stringify(importedData));
      // 更新游戏状态
      player.level = new OmegaNum( importedData.player.level);
      player.souls = new OmegaNum( importedData.player.souls);
      player.maxDefeatedLevel = new OmegaNum(importedData.player?.maxDefeatedLevel || 1);
      player.sacredSouls = new OmegaNum(importedData.player?.sacredSouls || 0);
      const upgrades = importedData.player?.upgrades || {};
      [1,2,3,4].forEach(n => {
        player.upgrades[n] = new OmegaNum(upgrades[n] || 1);
      });
      alert('从文本导入存档成功！');
    } else {
      alert('导入失败：存档数据格式不正确');
    }
  } catch (e) {
    alert('导入失败：无效的JSON格式');
  }
}

function importSave(event) {
  const file = event.target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = e => {
    try {
      const data = JSON.parse(e.target.result);
      if ((data.player?.level || data.playerLevel) && (data.player?.souls || data.playerSouls) !== undefined) {
        player.level = new OmegaNum(data.player?.level || data.playerLevel);
        player.souls = new OmegaNum(data.player?.souls || data.playerSouls);
        player.maxDefeatedLevel = new OmegaNum(data.player?.maxDefeatedLevel || 1);
        player.sacredSouls = new OmegaNum(data.player?.sacredSouls || 0);
        const upgrades = data.player?.upgrades || {};
        [1,2,3,4].forEach(n => {
          player.upgrades[n] = new OmegaNum(upgrades[n] || 1);
        });
        saveGame();
        alert('存档导入成功！');
      } else {
        alert('无效的存档文件');
      }
    } catch (err) {
      alert('导入失败：' + err.message);
    }
  };
  reader.readAsText(file);
}

function resetGameData() {
  if (confirm('确定要重置所有游戏数据吗？此操作不可恢复！')) {
    player.level = new OmegaNum(1);
    player.souls = new OmegaNum(0);
    player.maxDefeatedLevel = new OmegaNum(0);
    player.sacredSouls = new OmegaNum(0);
    [1,2,3,4].forEach(n => {
      player.upgrades[n] = new OmegaNum(1);
    });
    enemyLevel.value = new OmegaNum(1);
    enemyMaxLife.value = new OmegaNum(2);
    enemyCurrentLife.value = new OmegaNum(2);
    localStorage.removeItem('gameSave');
    saveData.value = null;
    saveDataText.value = '';
    alert('游戏数据已重置！');
  }
}

// 初始化时加载存档并启动自动保存
onMounted(() => {
  const saved = localStorage.getItem('gameSave');
  if (saved) {
    try {
      const data = JSON.parse(saved);
      player.level = new OmegaNum(data.player?.level || 1);
      player.souls = new OmegaNum(data.player?.souls || 0);
      player.maxDefeatedLevel = new OmegaNum(data.player?.maxDefeatedLevel || 1);
      player.sacredSouls = new OmegaNum(data.player?.sacredSouls || 0);
      const upgrades = data.player?.upgrades || {};
      [1,2,3,4].forEach(n => {
        player.upgrades[n] = new OmegaNum(upgrades[n] || 1);
      });
      saveData.value = data;
    } catch (err) {
      console.error('加载存档失败', err);
    }
  }
  // 启动每10秒自动保存
  autoSaveInterval.value = setInterval(() => {
    saveGame();
    console.log('自动保存成功');
    lastAutoSaveTime.value = new Date().toLocaleTimeString();
  }, 10000);
});

// 清理定时器和自动保存
onUnmounted(() => {
  clearInterval(autoSaveInterval.value);
  if (fightInterval.value) {
    clearInterval(fightInterval.value);
  }
});
</script>

<template>
  <div class="game-container">
    <h3>四位一体-quaternity</h3>
    <div class="tabs">
      <button 
        class="tab-button" 
        :class="{ active: currentTab === 'game' }"
        @click="currentTab = 'game'">
        主页
      </button>
      <button 
        class="tab-button" 
        @click="currentTab = 'soul'" 
        :class="{ active: currentTab === 'soul' }">
        炼魂
      </button>
      <button 
        class="tab-button" 
        :class="{ active: currentTab === 'save' }"
        @click="currentTab = 'save'">
        存档
      </button>
    </div>
    
    <!-- 游戏界面 -->
    <!-- 炼魂界面 -->
      <div v-if="currentTab === 'soul'" class="soul-tab">
        <button 
          @click="refineSouls"
          :disabled="!canRefine"
        >
          炼魂（获得{{ player.level.mul(player.maxDefeatedLevel).pow(0.5).format() }}圣魂）
        </button><br />
        {{canRefine ? '' : '炼魂需要玩家等级≥10且最高敌人等级≥10'}}
        （当前圣魂：{{ player.sacredSouls.format() }}）
        <div class="upgrades">
          <div class="upgrade-item">
            <p>升级1（等级{{ player.upgrades[1].format() }}）：每秒伤害乘以{{ player.upgrades[1].format() }}</p>
            <button @click="upgradeSacred(1)">
              升级（消耗：{{ OmegaNum.pow(2, player.upgrades[1]).format() }}圣魂）
            </button>
          </div>
          <div class="upgrade-item">
            <p>升级2（等级{{ player.upgrades[2].format() }}）：获得灵魂乘以{{ player.upgrades[2].format() }}</p>
            <button @click="upgradeSacred(2)">
              升级（消耗：{{ OmegaNum.pow(2, player.upgrades[2]).format() }}圣魂）
            </button>
          </div>
          <div class="upgrade-item">
            <p>升级3（等级{{ player.upgrades[3].format() }}）：敌人生命除以{{ player.upgrades[3].format() }}</p>
            <button @click="upgradeSacred(3)">
              升级（消耗：{{ OmegaNum.pow(2, player.upgrades[3]).format() }}圣魂）
            </button>
          </div>
          <div class="upgrade-item">
            <p>升级4（等级{{ player.upgrades[4].format() }}）：升级花费除以{{ player.upgrades[4].format() }}</p>
            <button @click="upgradeSacred(4)">
              升级（消耗：{{ OmegaNum.pow(2, player.upgrades[4]).format() }}圣魂）
            </button>
            </div>
          </div>
        </div>
      </div>

      <div v-if="currentTab === 'game'">
      <!-- 玩家信息 -->
      <div class="player-info">
      <h3>玩家状态</h3>
      <p>等级: {{ player.level.formatI() }}</p>
      <p>灵魂: {{ player.souls.format() }}</p>
      <button 
        @click="levelUp"
        :disabled="!player.souls.gte(totalLevelCost)"
      >
        升级（{{ totalLevelCost.format() }} 灵魂）
      </button>
    </div>
    
    <!-- 敌人选择 -->
    <div class="enemy-selection">
      <h3>选择敌人等级</h3>
      <p>当前选择: {{ enemyLevel }} 级</p>
      <div class="level-controls">
        <input
          type="range"
          min="1"
          :max="maxSelectableEnemyLevel"
          :value="enemyLevel"
          @input="selectEnemyLevel($event.target.valueAsNumber)"
          class="level-slider"
        >
        <input
          type="number"
          min="1"
          :max="maxSelectableEnemyLevel"
          :value="enemyLevel"
          @input="selectEnemyLevel($event.target.valueAsNumber)"
          class="level-input"
        >
      </div>
            <label class="continuous-checkbox">
        <input type="checkbox" v-model="isContinuous">
        持续战斗
      </label>
    </div>
    
    <!-- 战斗界面 -->
    <div class="battle" v-if="isFighting">
      <h3>战斗中...{{ isPaused ? '（已暂停）' : '' }}</h3>
      <p>等级: {{ enemyLevel.formatI() }}</p>
      <p>生命: {{ enemyCurrentLife.format() }} / {{ enemyMaxLife.format() }}</p>
      <div class="life-bar">
        <div 
          class="life-bar-fill"
          :style="{ width: lifeBarWidth }"
        ></div>
      </div>
      <p>每秒伤害: {{ totalDamage.format() }}</p>
      <p>掉落灵魂: {{ totalSoulGain.format() }}</p>
      <button @click="toggleFight">
        {{ isPaused ? '继续' : '暂停' }}
      </button>
  </div>
    
    <!-- 统计 -->
    <div class="history">
      <h3>统计</h3>
      <p>最高敌人等级: {{ player.maxDefeatedLevel.formatI() }}</p>
    </div>
    </div>

    <!-- 存档界面 -->
    <div v-if="currentTab === 'save'" class="save-interface">
  <div class="save-textarea">
    <textarea v-model="saveDataText" rows="10" placeholder="存档数据将显示在这里..."></textarea>
  </div>
      <h3>存档管理</h3>
      <div class="save-buttons">
        <button @click="saveGame" class="save-button">
          手动保存
        </button>
        <button @click="exportAsText" class="save-button">
      导出到输入框
        </button>
        <button @click="importFromText" class="save-button">
      从输入框导入
        </button>
        <button @click="exportSave" class="save-button">
      导出存档
        </button>
        <label class="save-button import-button">
          导入存档
          <input type="file" accept=".json" @change="importSave" hidden>
        </label>
        <button @click="resetGameData" class="save-button reset-button">
          重置游戏数据
        </button>
      </div>
    </div>
</template>

<style scoped>
.game-container {
  max-width: 1080px;
  min-width: 540px;
  margin: 0 auto;
  padding: 20px;
}

.tabs {
  display: flex;
  gap: 10px;
  margin: 20px 0;
}

.tab-button {
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  background-color: #444;
  color: white;
  cursor: pointer;
  transition: all 0.3s;
}

.tab-button.active {
  background-color: #6b46c1;
  font-weight: bold;
}

.tab-button:hover:not(.active) {
  background-color: #555;
}

.save-interface {
  background-color: rgba(255, 255, 255, 0.1);
  border-radius: 10px;
  padding: 20px;
  margin-top: 20px;
}

.save-textarea {
  margin-bottom: 15px;
}

.save-textarea textarea {
  width: 100%;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-family: monospace;
  resize: vertical;
}

.reset-button {
  background-color: #ff4444;
  margin-top: 10px;
}

.reset-button:hover {
  background-color: #cc0000;
}

.save-buttons {
  display: flex;
  gap: 15px;
  margin-bottom: 20px;
  flex-wrap: wrap;
}

.save-button {
  padding: 12px 24px;
  border: none;
  border-radius: 5px;
  background-color: #6b46c1;
  color: white;
  cursor: pointer;
  transition: background-color 0.3s;
  font-size: 16px;
}

.save-button:hover {
  background-color: #553c9a;
}

.import-button {
  display: inline-block;
}

.save-info {
  background-color: rgba(0, 0, 0, 0.2);
  padding: 15px;
  border-radius: 5px;
  line-height: 1.6;
}

.soul-tab {
  padding: 20px;
  button {
    margin: 10px;
    padding: 10px 20px;
    &[disabled] {
      opacity: 0.6;
      cursor: not-allowed;
    }
  }
  .upgrades {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 15px;
    .upgrade-item {
      border: 1px solid #555;
      padding: 10px;
      border-radius: 8px;
    }
  }
}

.player-info, .enemy-selection, .battle, .battle-prep, .history {
  background-color: rgba(255, 255, 255, 0.1);
  border-radius: 10px;
  padding: 20px;
  margin-bottom: 20px;
}

h1, h2, h3 {
  color: #42b883;
}

button {
  margin: 5px;
  padding: 10px 15px;
  background-color: #646cff;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
}

button:hover:not(:disabled) {
  background-color: #535bf2;
}

button:disabled {
  background-color: #666;
  cursor: not-allowed;
}

button.selected {
  background-color: #42b883;
}

.enemy-buttons {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
}

.life-bar {
  width: 100%;
  height: 20px;
  background-color: #333;
  border-radius: 10px;
  overflow: hidden;
  margin: 10px 0;
}

.life-bar-fill {
  height: 100%;
  background-color: #42b883;
  transition: width 0.5s;
}

.enemy-selection {
  margin-bottom: 20px;
  padding: 15px;
  background-color: rgba(255, 255, 255, 0.1);
  border-radius: 10px;
}

.continuous-checkbox {
  margin-left: 10px;
}

.battle-buttons {
  display: flex;
  gap: 10px;
  margin-top: 15px;
}

.level-controls {
  display: flex;
  align-items: center;
  gap: 10px;
  margin: 15px 0;
}

.level-slider {
  flex-grow: 1;
}

.level-input {
  width: 60px;
  padding: 5px;
}

.continuous-checkbox {
  margin-top: 10px;
  display: flex;
  align-items: center;
  gap: 5px;
}

p {
  margin: 8px 0;
}
</style>
