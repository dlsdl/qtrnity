<script setup>
import { ref, reactive, computed, onMounted, onUnmounted } from 'vue';
import OmegaNum from './OmegaNum.js';
import pako from 'pako';

// 游戏状态
const currentTab = ref('game');
// 圣魂系统状态迁移到player对象
const saveData = ref(null);
const saveDataText = ref('');
const isFighting = ref(true);
const isPaused = ref(true);

const player = reactive({
  // 自动化功能状态
  autoBuySacred: false,
  autoBuySacred2: false,
  autoBuySacred3: false,
  autoBuySacred4: false,
  autoGetSacred: false,
  autoGetSacred2: false,
  autoGetSacred3: false,
  autoGetSacred4: false,
  isContinuous: false,
  isAutoProgress: false,
  level: new OmegaNum(1),
  souls: new OmegaNum(0),
  experience: new OmegaNum(0),
  maxDefeatedLevel: new OmegaNum(1),
  sacredSouls: new OmegaNum(0),
  sacredSouls2: new OmegaNum(0),
  sacredSouls3: new OmegaNum(0),
  sacredSouls4: new OmegaNum(0),
  upgrades: {
    1: new OmegaNum(1),
    2: new OmegaNum(1),
    3: new OmegaNum(1),
    4: new OmegaNum(1),
  },
  upgrades2: {
    1: new OmegaNum(1),
    2: new OmegaNum(1),
    3: new OmegaNum(1),
    4: new OmegaNum(1),
  },
  upgrades3: {
    1: new OmegaNum(1),
    2: new OmegaNum(1),
    3: new OmegaNum(1),
    4: new OmegaNum(1),
  },
  upgrades4: {
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
const totalDamage=computed(()=> 
player.level
.pow(player.upgrades[1].gt(0) ? player.upgrades[1] : new OmegaNum(1))
.pow(player.upgrades2[1].gt(0) ? player.upgrades2[1] : new OmegaNum(1))
.pow(player.upgrades3[1].gt(0) ? player.upgrades3[1] : new OmegaNum(1))
.pow(player.upgrades4[1].gt(0) ? player.upgrades4[1] : new OmegaNum(1)));
const totalExpGain=computed(()=> 
enemyLevel.value
.pow(player.upgrades[2].gt(0) ? player.upgrades[2] : new OmegaNum(1))
.pow(player.upgrades2[2].gt(0) ? player.upgrades2[2] : new OmegaNum(1))
.pow(player.upgrades3[2].gt(0) ? player.upgrades3[2] : new OmegaNum(1))
.pow(player.upgrades4[2].gt(0) ? player.upgrades4[2] : new OmegaNum(1)));
const totalEnemyLife=computed(()=>
OmegaNum.pow(2, enemyLevel.value)
.root(player.upgrades[3].gt(0) ? player.upgrades[3] : new OmegaNum(1))
.root(player.upgrades2[3].gt(0) ? player.upgrades2[3] : new OmegaNum(1))
.root(player.upgrades3[3].gt(0) ? player.upgrades3[3] : new OmegaNum(1))
.root(player.upgrades4[3].gt(0) ? player.upgrades4[3] : new OmegaNum(1)));
const totalExpReq=computed(()=>
OmegaNum.pow(2, player.level)
.root(player.upgrades[4].gt(0) ? player.upgrades[4] : new OmegaNum(1))
.root(player.upgrades2[4].gt(0) ? player.upgrades2[4] : new OmegaNum(1))
.root(player.upgrades3[4].gt(0) ? player.upgrades3[4] : new OmegaNum(1))
.root(player.upgrades4[4].gt(0) ? player.upgrades4[4] : new OmegaNum(1)));
const expPercentage=computed(()=>{
  if (totalExpReq.value && totalExpReq.value.gt(0)) {
    return player.experience.div(totalExpReq.value).mul(100).toNumber();
  }
  return 0;
});
const maxSelectableEnemyLevel = computed(() => 
  new OmegaNum(player.maxDefeatedLevel).toNumber());
const skipLevelsInfo = computed(() => {
  const skipCount = calculateSkipLevels();
  return skipCount > 0 ? `可跳过${skipCount}个等级` : '';
});
const canRefine = computed(() => 
player.level.gte(10) && player.maxDefeatedLevel.gte(10));
const canRefine2 = computed(() => 
player.level.gte(1000) && player.maxDefeatedLevel.gte(1000));
const canRefine3 = computed(() => 
player.level.gte(1e7) && player.maxDefeatedLevel.gte(1e7));
const canRefine4 = computed(() => 
player.level.gte(1e13) && player.maxDefeatedLevel.gte(1e13));
const soulGain = computed(() => 
OmegaNum.pow(1e12,player.level.mul(player.maxDefeatedLevel).div(100).pow(1/16).sub(1)).min("e9e15"));
const soulGain2 = computed(() => 
OmegaNum.pow(1e6,player.level.mul(player.maxDefeatedLevel).div(1e6).pow(1/16).sub(1)).min("e9e15"));
const soulGain3 = computed(() => 
OmegaNum.pow(1e4,player.level.mul(player.maxDefeatedLevel).div(1e14).pow(1/16).sub(1)).min("e9e15"));
const soulGain4 = computed(() => 
OmegaNum.pow(1e3,player.level.mul(player.maxDefeatedLevel).div(1e26).pow(1/16).sub(1)).min("e9e15"));

// 自动化功能解锁条件
const canAutoBuySacred = computed(() => player.sacredSouls.gte(1e6));
const canAutoBuySacred2 = computed(() => player.sacredSouls2.gte(1e6));
const canAutoBuySacred3 = computed(() => player.sacredSouls3.gte(1e6));
const canAutoBuySacred4 = computed(() => player.sacredSouls4.gte(1e6));

const canAutoGetSacred = computed(() => player.sacredSouls.gte(1e12));
const canAutoGetSacred2 = computed(() => player.sacredSouls2.gte(1e12));
const canAutoGetSacred3 = computed(() => player.sacredSouls3.gte(1e12));
const canAutoGetSacred4 = computed(() => player.sacredSouls4.gte(1e12));

const canUnlockContinuous = computed(() => player.level.gte(5));
const canUnlockAutoProgress = computed(() => player.level.gte(25));
const expPerSecond = computed(() => {
  // 计算击败敌人所需时间（毫秒）
  const timeToDefeatMs = (enemyMaxLife.value.div(totalDamage.value.div(10))).mul(100).add(100);
  // 转换为秒
  const timeToDefeatSeconds = timeToDefeatMs.div(1000);
  // 每秒经验 = 总经验 / 时间（秒）
  return totalExpGain.value.div(timeToDefeatSeconds);
});
const lifeBarWidth = computed(() => {
  if(enemyCurrentLife.value && enemyMaxLife.value)
    return (enemyCurrentLife.value.div(enemyMaxLife.value)).toNumber() * 100 + '%';
  return '0%';
});

// 炼魂功能
function refineSouls() {
  player.sacredSouls = player.sacredSouls.add(soulGain.value);
  player.level = new OmegaNum(1);
  player.maxDefeatedLevel = new OmegaNum(1);
  player.souls = new OmegaNum(0);
  player.experience = new OmegaNum(0);
  enemyLevel.value = new OmegaNum(1);
  enemyMaxLife.value = totalEnemyLife.value;
  enemyCurrentLife.value = new OmegaNum(enemyMaxLife.value);
}

// 炼魂^2重置功能
function refineSouls2() {
  player.sacredSouls2 = player.sacredSouls2.add(soulGain2.value);
  player.level = new OmegaNum(1);
  player.maxDefeatedLevel = new OmegaNum(1);
  player.souls = new OmegaNum(0);
  player.experience = new OmegaNum(0);
  player.sacredSouls = new OmegaNum(0);
  // 重置圣魂升级
  [1,2,3,4].forEach(n => {
    player.upgrades[n] = new OmegaNum(1);
  });
  enemyLevel.value = new OmegaNum(1);
  enemyMaxLife.value = totalEnemyLife.value;
  enemyCurrentLife.value = new OmegaNum(enemyMaxLife.value);
}

// 炼魂^3重置功能
function refineSouls3() {
  player.sacredSouls3 = player.sacredSouls3.add(soulGain3.value);
  player.level = new OmegaNum(1);
  player.maxDefeatedLevel = new OmegaNum(1);
  player.souls = new OmegaNum(0);
  player.experience = new OmegaNum(0);
  player.sacredSouls = new OmegaNum(0);
  player.sacredSouls2 = new OmegaNum(0);
  // 重置圣魂和圣^2魂升级
  [1,2,3,4].forEach(n => {
    player.upgrades[n] = new OmegaNum(1);
    player.upgrades2[n] = new OmegaNum(1);
  });
  enemyLevel.value = new OmegaNum(1);
  enemyMaxLife.value = totalEnemyLife.value;
  enemyCurrentLife.value = new OmegaNum(enemyMaxLife.value);
}

// 炼魂^4重置功能
function refineSouls4() {
  player.sacredSouls4 = player.sacredSouls4.add(soulGain4.value);
  player.level = new OmegaNum(1);
  player.maxDefeatedLevel = new OmegaNum(1);
  player.souls = new OmegaNum(0);
  player.experience = new OmegaNum(0);
  player.sacredSouls = new OmegaNum(0);
  player.sacredSouls2 = new OmegaNum(0);
  player.sacredSouls3 = new OmegaNum(0);
  // 重置圣魂、圣^2魂和圣^3魂升级
  [1,2,3,4].forEach(n => {
    player.upgrades[n] = new OmegaNum(1);
    player.upgrades2[n] = new OmegaNum(1);
    player.upgrades3[n] = new OmegaNum(1);
  });
  enemyLevel.value = new OmegaNum(1);
  enemyMaxLife.value = totalEnemyLife.value;
  enemyCurrentLife.value = new OmegaNum(enemyMaxLife.value);
}

// 圣魂升级功能
function upgradeSacred(type) {
  let cost = OmegaNum.pow(16, player.upgrades[type].sub(1));
  if (player.sacredSouls.gte(cost)) {
    player.sacredSouls = player.sacredSouls.sub(cost);
    player.upgrades[type] = player.upgrades[type].add(1);
  }
}

// 圣^2魂升级功能
function upgradeSacred2(type) {
  let cost = OmegaNum.pow(16, player.upgrades2[type].sub(1));
  if (player.sacredSouls2.gte(cost)) {
    player.sacredSouls2 = player.sacredSouls2.sub(cost);
    player.upgrades2[type] = player.upgrades2[type].add(1);
  }
}

// 圣^3魂升级功能
function upgradeSacred3(type) {
  let cost = OmegaNum.pow(16, player.upgrades3[type].sub(1));
  if (player.sacredSouls3.gte(cost)) {
    player.sacredSouls3 = player.sacredSouls3.sub(cost);
    player.upgrades3[type] = player.upgrades3[type].add(1);
  }
}

// 圣^4魂升级功能
function upgradeSacred4(type) {
  let cost = OmegaNum.pow(16, player.upgrades4[type].sub(1));
  if (player.sacredSouls4.gte(cost)) {
    player.sacredSouls4 = player.sacredSouls4.sub(cost);
    player.upgrades4[type] = player.upgrades4[type].add(1);
  }
}

// 检查经验升级（优化版：使用等比数列求和公式直接计算可升级次数）
function checkLevelUp() {
  while (true) {
    // 计算当前等级的经验需求
    const currentExpReq = totalExpReq.value;
    if (player.experience.lt(currentExpReq)) break;
    
    // 计算经验升级的等比数列参数
    const root1 = player.upgrades[4].gt(0) ? player.upgrades[4] : new OmegaNum(1);
    const root2 = player.upgrades2[4].gt(0) ? player.upgrades2[4] : new OmegaNum(1);
    const root3 = player.upgrades3[4].gt(0) ? player.upgrades3[4] : new OmegaNum(1);
    const root4 = player.upgrades4[4].gt(0) ? player.upgrades4[4] : new OmegaNum(1);
    const totalRoot = root1.mul(root2).mul(root3).mul(root4);
    
    // 等比数列参数：经验需求是 2^level / totalRoot，每次升级level+1，所以公比是 2^(1/totalRoot)
    const r = OmegaNum.pow(2, 1).root(totalRoot); // 公比
    
    if (r.eq(1)) {
      // 特殊情况：公比为1，每次经验需求相同
      const levelsGained = player.experience.div(currentExpReq).floor();
      if (levelsGained.lte(0)) break;
      
      player.experience = player.experience.sub(currentExpReq.mul(levelsGained));
      player.level = player.level.add(levelsGained);
      break;
    }
    
    // 计算剩余经验能升级多少次
    // 等比数列求和：S = a1 * (r^n - 1) / (r - 1) ≤ availableExp
    const availableExp = player.experience;
    const a1 = currentExpReq; // 首项（当前等级的经验需求）
    
    // 重排公式：r^n ≤ (availableExp * (r - 1) / a1) + 1
    const rMinus1 = r.sub(1);
    const rightSide = availableExp.mul(rMinus1).div(a1).add(1);
    
    if (rightSide.lte(1)) break;
    
    // 使用logBase直接计算n
    const lnRightSide = rightSide.logBase(Math.E); // 自然对数
    const lnR = r.logBase(Math.E); // 公比的自然对数
    const n = lnRightSide.div(lnR).floor();
    
    if (n.lte(0)) break;
    
    // 计算实际消耗的经验
    // 使用等比数列求和公式计算前n项和
    const totalExpCost = a1.mul(r.pow(n).sub(1)).div(rMinus1);
    
    // 更新玩家经验和等级
    player.experience = player.experience.sub(totalExpCost);
    player.level = player.level.add(n);
  }
}

// 计算可以跳过的等级数量（优化版：使用等比数列求和公式和logBase方法）
function calculateSkipLevels() {
  // 使用每秒伤害除以10，因为战斗循环每0.1秒执行一次
  const damagePerTick = totalDamage.value.div(10);
  if (damagePerTick.lte(0)) return 0;
  
  let currentLevel = enemyLevel.value.toNumber();
  
  // 计算总根值：敌人生命的总分母
  const root1 = player.upgrades[3].gt(0) ? player.upgrades[3] : new OmegaNum(1);
  const root2 = player.upgrades2[3].gt(0) ? player.upgrades2[3] : new OmegaNum(1);
  const root3 = player.upgrades3[3].gt(0) ? player.upgrades3[3] : new OmegaNum(1);
  const root4 = player.upgrades4[3].gt(0) ? player.upgrades4[3] : new OmegaNum(1);
  const totalRoot = root1.mul(root2).mul(root3).mul(root4);
  
  // 等比数列参数
  const firstLevel = currentLevel + 1;
  const a1 = OmegaNum.pow(2, firstLevel).root(totalRoot); // 首项
  const r = OmegaNum.pow(2, 1).root(totalRoot); // 公比 = 2^(1/totalRoot)
  
  // 如果公比小于等于1，无法无限增长，直接返回0
  if (r.lte(1)) return 0;
  
  // 等比数列求和公式：S = a1 * (r^n - 1) / (r - 1) ≤ damagePerTick
  // 解这个不等式求n
  
  // 首先计算 r - 1
  const rMinus1 = r.sub(1);
  if (rMinus1.lte(0)) return 0;
  
  // 重排公式：r^n ≤ (damagePerTick * (r - 1) / a1) + 1
  const rightSide = damagePerTick.mul(rMinus1).div(a1).add(1);
  if (rightSide.lte(1)) return 0;
  
  // 使用OmegaNum的logBase方法直接计算n
  // log_r(rightSide) = ln(rightSide) / ln(r)
  const lnRightSide = rightSide.logBase(Math.E); // 自然对数
  const lnR = r.logBase(Math.E); // 公比的自然对数
  let n = lnRightSide.div(lnR).floor().toNumber();
  
  // 确保n是非负整数
  n = Math.max(1, n);
  
  // 验证计算结果，确保总和不超过伤害
  //let totalLife = new OmegaNum(0);
  //let actualSkipCount = 0;
  //for (let i = 0; i < n; i++) {
  //  const nextLevelLife = OmegaNum.pow(2, currentLevel + i + 1).root(totalRoot);
  //  totalLife = totalLife.add(nextLevelLife);
  //  if (totalLife.gt(damagePerTick)) {
  //    break;
  //  }
  //  actualSkipCount++;
  //}
  
  return n-1;
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
      // 获得经验奖励
      player.experience = player.experience.add(totalExpGain.value);
      
      // 检查升级
      checkLevelUp();
      
      // 计算跳关等级数量
      const skipCount = calculateSkipLevels();
      
      // 更新最高击败等级
      const newMaxLevel = enemyLevel.value.add(1 + skipCount);
      if (newMaxLevel.gte(player.maxDefeatedLevel)) {
        player.maxDefeatedLevel = newMaxLevel;
      }
      
      // 自动推关逻辑
      if (player.isAutoProgress) {
        // 敌人等级增加跳过的等级数量
        const newLevel = enemyLevel.value.add(1).add(skipCount);
        selectEnemyLevel(newLevel);
      }
      
      // 重置敌人生命
      enemyMaxLife.value = totalEnemyLife.value;
      enemyCurrentLife.value = enemyMaxLife.value.clone();
      
      // 持续战斗逻辑
      if (player.isContinuous) {

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
      experience: player.experience.toString(),
      maxDefeatedLevel: player.maxDefeatedLevel.toString(),
      sacredSouls: player.sacredSouls.toString(),
      sacredSouls2: player.sacredSouls2.toString(),
      sacredSouls3: player.sacredSouls3.toString(),
      sacredSouls4: player.sacredSouls4.toString(),
      // 自动化功能状态
      autoBuySacred: player.autoBuySacred,
      autoBuySacred2: player.autoBuySacred2,
      autoBuySacred3: player.autoBuySacred3,
      autoBuySacred4: player.autoBuySacred4,
      autoGetSacred: player.autoGetSacred,
      autoGetSacred2: player.autoGetSacred2,
      autoGetSacred3: player.autoGetSacred3,
      autoGetSacred4: player.autoGetSacred4,
      isContinuous: player.isContinuous,
      isAutoProgress: player.isAutoProgress,
      upgrades: {
        1: player.upgrades[1].toString(),
        2: player.upgrades[2].toString(),
        3: player.upgrades[3].toString(),
        4: player.upgrades[4].toString()
      },
      upgrades2: {
        1: player.upgrades2[1].toString(),
        2: player.upgrades2[2].toString(),
        3: player.upgrades2[3].toString(),
        4: player.upgrades2[4].toString()
      },
      upgrades3: {
        1: player.upgrades3[1].toString(),
        2: player.upgrades3[2].toString(),
        3: player.upgrades3[3].toString(),
        4: player.upgrades3[4].toString()
      },
      upgrades4: {
        1: player.upgrades4[1].toString(),
        2: player.upgrades4[2].toString(),
        3: player.upgrades4[3].toString(),
        4: player.upgrades4[4].toString()
      }
    }
  };
  localStorage.setItem('gameSave', JSON.stringify(data));
  saveData.value = data;
}

function exportAsText(){
    const dataToExport = saveData.value || {
        player: {
          level: player.level,
          souls: player.souls,
          experience: player.experience,
          maxDefeatedLevel: player.maxDefeatedLevel,
          sacredSouls: player.sacredSouls,
          sacredSouls2: player.sacredSouls2,
          sacredSouls3: player.sacredSouls3,
          sacredSouls4: player.sacredSouls4,
          autoBuySacred: player.autoBuySacred,
          autoBuySacred2: player.autoBuySacred2,
          autoBuySacred3: player.autoBuySacred3,
          autoBuySacred4: player.autoBuySacred4,
          autoGetSacred: player.autoGetSacred,
          autoGetSacred2: player.autoGetSacred2,
          autoGetSacred3: player.autoGetSacred3,
          autoGetSacred4: player.autoGetSacred4,
          isContinuous: player.isContinuous,
          isAutoProgress: player.isAutoProgress,
          upgrades: {
            1: player.upgrades[1],
            2: player.upgrades[2],
            3: player.upgrades[3],
            4: player.upgrades[4]
          },
          upgrades2: {
            1: player.upgrades2[1],
            2: player.upgrades2[2],
            3: player.upgrades2[3],
            4: player.upgrades2[4]
          },
          upgrades3: {
            1: player.upgrades3[1],
            2: player.upgrades3[2],
            3: player.upgrades3[3],
            4: player.upgrades3[4]
          },
          upgrades4: {
            1: player.upgrades4[1],
            2: player.upgrades4[2],
            3: player.upgrades4[3],
            4: player.upgrades4[4]
          }
        }
      };
    
    // 转换为JSON字符串
    const jsonString = JSON.stringify(dataToExport);
    // 使用pako压缩为二进制数据
    const compressed = pako.deflate(jsonString);
    // 转换为base64格式
    const base64 = btoa(String.fromCharCode.apply(null, compressed));
    
    saveDataText.value = base64;
}

function exportSave() {
  const dataToExport = saveData.value || {
    player: {
          level: player.level,
          souls: player.souls,
          experience: player.experience,
          maxDefeatedLevel: player.maxDefeatedLevel,
          sacredSouls: player.sacredSouls,
          sacredSouls2: player.sacredSouls2,
          sacredSouls3: player.sacredSouls3,
          sacredSouls4: player.sacredSouls4,
          autoBuySacred: player.autoBuySacred,
          autoBuySacred2: player.autoBuySacred2,
          autoBuySacred3: player.autoBuySacred3,
          autoBuySacred4: player.autoBuySacred4,
          autoGetSacred: player.autoGetSacred,
          autoGetSacred2: player.autoGetSacred2,
          autoGetSacred3: player.autoGetSacred3,
          autoGetSacred4: player.autoGetSacred4,
          isContinuous: player.isContinuous,
          isAutoProgress: player.isAutoProgress,
          upgrades: {
            1: player.upgrades[1],
            2: player.upgrades[2],
            3: player.upgrades[3],
            4: player.upgrades[4]
          },
          upgrades2: {
            1: player.upgrades2[1],
            2: player.upgrades2[2],
            3: player.upgrades2[3],
            4: player.upgrades2[4]
          },
          upgrades3: {
            1: player.upgrades3[1],
            2: player.upgrades3[2],
            3: player.upgrades3[3],
            4: player.upgrades3[4]
          },
          upgrades4: {
            1: player.upgrades4[1],
            2: player.upgrades4[2],
            3: player.upgrades4[3],
            4: player.upgrades4[4]
          }
        }
  };
  
  // 转换为JSON字符串
  const jsonString = JSON.stringify(dataToExport);
  // 使用pako压缩为二进制数据
  const compressed = pako.deflate(jsonString);
  // 转换为base64格式
  const base64 = btoa(String.fromCharCode.apply(null, compressed));

  const blob = new Blob([base64], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `qtrnity-save-${new Date().toISOString().slice(0,10)}.json`;
  a.click();
  URL.revokeObjectURL(url);
}

function importFromText() {
  try {
    let importedData;
    
    // 尝试处理压缩后的base64数据
    try {
      // 解码base64字符串
      const base64Decoded = atob(saveDataText.value);
      // 转换为Uint8Array用于pako解压缩
      const uint8Array = new Uint8Array(base64Decoded.length);
      for (let i = 0; i < base64Decoded.length; i++) {
        uint8Array[i] = base64Decoded.charCodeAt(i);
      }
      // 使用pako解压缩
      const decompressed = pako.inflate(uint8Array, { to: 'string' });
      // 解析JSON
      importedData = JSON.parse(decompressed);
    } catch (compressionError) {
      // 如果处理压缩数据失败，尝试直接解析JSON（兼容旧存档）
      importedData = JSON.parse(saveDataText.value);
    }
    
    // 验证存档数据结构
    if ((importedData.player?.level || importedData.playerLevel) && (importedData.player?.souls || importedData.playerSouls)) {
      saveData.value = importedData;
      localStorage.setItem('gameSave', JSON.stringify(importedData));
      // 更新游戏状态
      player.level = new OmegaNum( importedData.player.level);
      player.souls = new OmegaNum( importedData.player.souls);
      player.experience = new OmegaNum( importedData.player.experience || 0);
      player.maxDefeatedLevel = new OmegaNum(importedData.player?.maxDefeatedLevel || 1);
      player.sacredSouls = new OmegaNum(importedData.player?.sacredSouls || 0);
      player.sacredSouls2 = new OmegaNum(importedData.player?.sacredSouls2 || 0);
      player.sacredSouls3 = new OmegaNum(importedData.player?.sacredSouls3 || 0);
      player.sacredSouls4 = new OmegaNum(importedData.player?.sacredSouls4 || 0);
      
      // 加载自动化功能状态
      player.autoBuySacred = importedData.player?.autoBuySacred || false;
      player.autoBuySacred2 = importedData.player?.autoBuySacred2 || false;
      player.autoBuySacred3 = importedData.player?.autoBuySacred3 || false;
      player.autoBuySacred4 = importedData.player?.autoBuySacred4 || false;
      player.autoGetSacred = importedData.player?.autoGetSacred || false;
      player.autoGetSacred2 = importedData.player?.autoGetSacred2 || false;
      player.autoGetSacred3 = importedData.player?.autoGetSacred3 || false;
      player.autoGetSacred4 = importedData.player?.autoGetSacred4 || false;
      player.isContinuous = importedData.player?.isContinuous || false;
      player.isAutoProgress = importedData.player?.isAutoProgress || false;
      
      const upgrades = importedData.player?.upgrades || {};
      [1,2,3,4].forEach(n => {
        player.upgrades[n] = new OmegaNum(upgrades[n] || 1);
      });
      const upgrades2 = importedData.player?.upgrades2 || {};
      [1,2,3,4].forEach(n => {
        player.upgrades2[n] = new OmegaNum(upgrades2[n] || 1);
      });
      const upgrades3 = importedData.player?.upgrades3 || {};
      [1,2,3,4].forEach(n => {
        player.upgrades3[n] = new OmegaNum(upgrades3[n] || 1);
      });
      const upgrades4 = importedData.player?.upgrades4 || {};
      [1,2,3,4].forEach(n => {
        player.upgrades4[n] = new OmegaNum(upgrades4[n] || 1);
      });
      alert('从文本导入存档成功！');
    } else {
      alert('导入失败：存档数据格式不正确');
    }
  } catch (e) {
    alert('导入失败：' + e.message);
  }
}

function importSave(event) {
  const file = event.target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = e => {
    try {
      let data;
      const fileContent = e.target.result;
      
      // 尝试处理压缩后的base64数据
    try {
      // 解码base64字符串
      const base64Decoded = atob(fileContent);
      // 转换为Uint8Array用于pako解压缩
      const uint8Array = new Uint8Array(base64Decoded.length);
      for (let i = 0; i < base64Decoded.length; i++) {
        uint8Array[i] = base64Decoded.charCodeAt(i);
      }
      // 使用pako解压缩
      const decompressed = pako.inflate(uint8Array, { to: 'string' });
      // 解析JSON
      data = JSON.parse(decompressed);
    } catch (compressionError) {
      // 如果处理压缩数据失败，尝试直接解析JSON（兼容旧存档）
      data = JSON.parse(fileContent);
    }
      
      if ((data.player?.level || data.playerLevel) && (data.player?.souls || data.playerSouls) !== undefined) {
        player.level = new OmegaNum(data.player?.level || data.playerLevel);
        player.souls = new OmegaNum(data.player?.souls || data.playerSouls);
        player.experience = new OmegaNum(data.player?.experience || 0);
        player.maxDefeatedLevel = new OmegaNum(data.player?.maxDefeatedLevel || 1);
        player.sacredSouls = new OmegaNum(data.player?.sacredSouls || 0);
        player.sacredSouls2 = new OmegaNum(data.player?.sacredSouls2 || 0);
      player.sacredSouls3 = new OmegaNum(data.player?.sacredSouls3 || 0);
      player.sacredSouls4 = new OmegaNum(data.player?.sacredSouls4 || 0);
      
      // 加载自动化功能状态
      player.autoBuySacred = data.player?.autoBuySacred || false;
      player.autoBuySacred2 = data.player?.autoBuySacred2 || false;
      player.autoBuySacred3 = data.player?.autoBuySacred3 || false;
      player.autoBuySacred4 = data.player?.autoBuySacred4 || false;
      player.autoGetSacred = data.player?.autoGetSacred || false;
      player.autoGetSacred2 = data.player?.autoGetSacred2 || false;
      player.autoGetSacred3 = data.player?.autoGetSacred3 || false;
      player.autoGetSacred4 = data.player?.autoGetSacred4 || false;
      player.isContinuous = data.player?.isContinuous || false;
      player.isAutoProgress = data.player?.isAutoProgress || false;
      
      const upgrades = data.player?.upgrades || {};
      [1,2,3,4].forEach(n => {
        player.upgrades[n] = new OmegaNum(upgrades[n] || 1);
      });
      const upgrades2 = data.player?.upgrades2 || {};
      [1,2,3,4].forEach(n => {
        player.upgrades2[n] = new OmegaNum(upgrades2[n] || 1);
      });
      const upgrades3 = data.player?.upgrades3 || {};
      [1,2,3,4].forEach(n => {
        player.upgrades3[n] = new OmegaNum(upgrades3[n] || 1);
      });
      const upgrades4 = data.player?.upgrades4 || {};
      [1,2,3,4].forEach(n => {
        player.upgrades4[n] = new OmegaNum(upgrades4[n] || 1);
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
    player.experience = new OmegaNum(0);
    player.maxDefeatedLevel = new OmegaNum(0);
    player.sacredSouls = new OmegaNum(0);
    player.sacredSouls2 = new OmegaNum(0);
    player.sacredSouls3 = new OmegaNum(0);
    player.sacredSouls4 = new OmegaNum(0);
    player.autoBuySacred = false;
    player.autoBuySacred2 = false;
    player.autoBuySacred3 = false;
    player.autoBuySacred4 = false;
    player.autoGetSacred = false;
    player.autoGetSacred2 = false;
    player.autoGetSacred3 = false;
    player.autoGetSacred4 = false;
    player.isContinuous = false;
    player.isAutoProgress = false;
    [1,2,3,4].forEach(n => {
      player.upgrades[n] = new OmegaNum(1);
    });
    [1,2,3,4].forEach(n => {
      player.upgrades2[n] = new OmegaNum(1);
    });
    [1,2,3,4].forEach(n => {
      player.upgrades3[n] = new OmegaNum(1);
    });
    [1,2,3,4].forEach(n => {
      player.upgrades4[n] = new OmegaNum(1);
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

// 自动化功能定时器
const autoFunctionInterval = ref(null);

// 自动购买升级（优化版：一次升最大等级）
function autoBuyUpgrades() {
  // 等比数列求和函数：计算从当前等级到最大等级所需的总魂数
  // 升级成本是等比数列：第k级成本 = base^(k-1)，其中base=16
  // 等比数列求和公式：S = a1 * (r^n - 1) / (r - 1) （r≠1）
  function calculateTotalCost(currentLevel, maxLevel, base) {
    const startLevel = currentLevel.toNumber();
    const endLevel = maxLevel.toNumber();
    const terms = endLevel - startLevel;
    
    if (terms <= 0) return new OmegaNum(0);
    
    const r = new OmegaNum(base); // 公比
    const a1 = OmegaNum.pow(base, startLevel - 1); // 首项：第startLevel级的升级成本
    
    // 使用等比数列求和公式
    if (r.eq(1)) {
      // 特殊情况：公比为1，所有项相等
      return a1.mul(terms);
    } else {
      // 正常情况：S = a1 * (r^terms - 1) / (r - 1)
      const rPowerTerms = r.pow(terms);
      const numerator = rPowerTerms.sub(1);
      const denominator = r.sub(1);
      return a1.mul(numerator).div(denominator);
    }
  }
  
  // 自动购买圣魂升级
  if (player.autoBuySacred) {
    for (let type = 1; type <= 4; type++) {
      // 计算最大等级：最大等级 = 魂的数量.logBase(16).add(2)
      const maxLevel = player.sacredSouls.logBase(16).add(2).floor();
      const currentLevel = player.upgrades[type];
      
      if (maxLevel.gt(currentLevel)) {
        // 使用等比数列公式计算总魂数
        const totalCost = calculateTotalCost(currentLevel, maxLevel, 16);
        
        // 如果魂的数量足够，一次性升级到最大等级
        if (player.sacredSouls.gte(totalCost)) {
          player.sacredSouls = player.sacredSouls.sub(totalCost);
          player.upgrades[type] = maxLevel;
        }
      }
    }
  }
  
  // 自动购买圣^2魂升级
  if (player.autoBuySacred2) {
    for (let type = 1; type <= 4; type++) {
      // 计算最大等级：最大等级 = 魂的数量.logBase(16).add(2)
      const maxLevel = player.sacredSouls2.logBase(16).add(2).floor();
      const currentLevel = player.upgrades2[type];
      
      if (maxLevel.gt(currentLevel)) {
        // 使用等比数列公式计算总魂数
        const totalCost = calculateTotalCost(currentLevel, maxLevel, 16);
        
        // 如果魂的数量足够，一次性升级到最大等级
        if (player.sacredSouls2.gte(totalCost)) {
          player.sacredSouls2 = player.sacredSouls2.sub(totalCost);
          player.upgrades2[type] = maxLevel;
        }
      }
    }
  }
  
  // 自动购买圣^3魂升级
  if (player.autoBuySacred3) {
    for (let type = 1; type <= 4; type++) {
      // 计算最大等级：最大等级 = 魂的数量.logBase(16).add(2)
      const maxLevel = player.sacredSouls3.logBase(16).add(2).floor();
      const currentLevel = player.upgrades3[type];
      
      if (maxLevel.gt(currentLevel)) {
        // 使用等比数列公式计算总魂数
        const totalCost = calculateTotalCost(currentLevel, maxLevel, 16);
        
        // 如果魂的数量足够，一次性升级到最大等级
        if (player.sacredSouls3.gte(totalCost)) {
          player.sacredSouls3 = player.sacredSouls3.sub(totalCost);
          player.upgrades3[type] = maxLevel;
        }
      }
    }
  }
  
  // 自动购买圣^4魂升级
  if (player.autoBuySacred4) {
    for (let type = 1; type <= 4; type++) {
      // 计算最大等级：最大等级 = 魂的数量.logBase(16).add(2)
      const maxLevel = player.sacredSouls4.logBase(16).add(2).floor();
      const currentLevel = player.upgrades4[type];
      
      if (maxLevel.gt(currentLevel)) {
        // 使用等比数列公式计算总魂数
        const totalCost = calculateTotalCost(currentLevel, maxLevel, 16);
        
        // 如果魂的数量足够，一次性升级到最大等级
        if (player.sacredSouls4.gte(totalCost)) {
          player.sacredSouls4 = player.sacredSouls4.sub(totalCost);
          player.upgrades4[type] = maxLevel;
        }
      }
    }
  }
}

// 自动获得资源
function autoGetResources() {
  if (player.autoGetSacred) {
    player.sacredSouls = player.sacredSouls.add(soulGain.value.div(10)); // 每100ms获得1/10
  }
  
  if (player.autoGetSacred2) {
    player.sacredSouls2 = player.sacredSouls2.add(soulGain2.value.div(10)); // 每100ms获得1/10
  }
  
  if (player.autoGetSacred3) {
    player.sacredSouls3 = player.sacredSouls3.add(soulGain3.value.div(10)); // 每100ms获得1/10
  }
  
  if (player.autoGetSacred4) {
    player.sacredSouls4 = player.sacredSouls4.add(soulGain4.value.div(10)); // 每100ms获得1/10
  }
}

// 自动执行功能
function runAutoFunctions() {
  autoBuyUpgrades();
  autoGetResources();
}

// 初始化时加载存档并启动自动保存
onMounted(() => {
  const saved = localStorage.getItem('gameSave');
  if (saved) {
    try {
      const data = JSON.parse(saved);
      player.level = new OmegaNum(data.player?.level || 1);
      player.souls = new OmegaNum(data.player?.souls || 0);
      player.experience = new OmegaNum(data.player?.experience || 0);
      player.maxDefeatedLevel = new OmegaNum(data.player?.maxDefeatedLevel || 1);
      player.sacredSouls = new OmegaNum(data.player?.sacredSouls || 0);
      player.sacredSouls2 = new OmegaNum(data.player?.sacredSouls2 || 0);
      player.sacredSouls3 = new OmegaNum(data.player?.sacredSouls3 || 0);
      player.sacredSouls4 = new OmegaNum(data.player?.sacredSouls4 || 0);
      
      // 加载自动化功能状态
      player.autoBuySacred = data.player?.autoBuySacred || false;
      player.autoBuySacred2 = data.player?.autoBuySacred2 || false;
      player.autoBuySacred3 = data.player?.autoBuySacred3 || false;
      player.autoBuySacred4 = data.player?.autoBuySacred4 || false;
      player.autoGetSacred = data.player?.autoGetSacred || false;
      player.autoGetSacred2 = data.player?.autoGetSacred2 || false;
      player.autoGetSacred3 = data.player?.autoGetSacred3 || false;
      player.autoGetSacred4 = data.player?.autoGetSacred4 || false;
      player.isContinuous = data.player?.isContinuous || false;
      player.isAutoProgress = data.player?.isAutoProgress || false;
      
      const upgrades = data.player?.upgrades || {};
      [1,2,3,4].forEach(n => {
        player.upgrades[n] = new OmegaNum(upgrades[n] || 1);
      });
      const upgrades2 = data.player?.upgrades2 || {};
      [1,2,3,4].forEach(n => {
        player.upgrades2[n] = new OmegaNum(upgrades2[n] || 1);
      });
      const upgrades3 = data.player?.upgrades3 || {};
      [1,2,3,4].forEach(n => {
        player.upgrades3[n] = new OmegaNum(upgrades3[n] || 1);
      });
      const upgrades4 = data.player?.upgrades4 || {};
      [1,2,3,4].forEach(n => {
        player.upgrades4[n] = new OmegaNum(upgrades4[n] || 1);
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
  
  // 启动自动化功能定时器（每100ms执行一次）
  autoFunctionInterval.value = setInterval(() => {
    runAutoFunctions();
  }, 100);
});

// 清理定时器和自动保存
onUnmounted(() => {
  clearInterval(autoSaveInterval.value);
  clearInterval(autoFunctionInterval.value);
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
        :class="{ active: currentTab === 'auto' }"
        @click="currentTab = 'auto'">
        自动
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
      <h3 style="margin-top: 30px; color: #ff6b6b;">炼魂系统</h3>
      <button @click="refineSouls" :disabled="!canRefine">
          炼魂重置（获得{{ soulGain.format() }}圣魂）
      </button><br />
      {{canRefine ? '' : '炼魂需要玩家等级≥10且最高敌人等级≥10'}}
      （当前圣魂：{{ player.sacredSouls.format() }}）
      <div class="upgrades" style="margin-top: 20px;">
        <div class="upgrade-item">
          <p>升级1（等级{{ player.upgrades[1].format() }}）：每秒伤害^{{ player.upgrades[1].format() }}</p>
          <button @click="upgradeSacred(1)">
              升级（消耗：{{ OmegaNum.pow(16, player.upgrades[1].sub(1)).format() }}圣魂）
          </button>
        </div>
        <div class="upgrade-item">
          <p>升级2（等级{{ player.upgrades[2].format() }}）：获得经验^{{ player.upgrades[2].format() }}</p>
          <button @click="upgradeSacred(2)">
            升级（消耗：{{ OmegaNum.pow(16, player.upgrades[2].sub(1)).format() }}圣魂）
          </button>
        </div>
        <div class="upgrade-item">
          <p>升级3（等级{{ player.upgrades[3].format() }}）：敌人生命^1/{{ player.upgrades[3].format() }}</p>
          <button @click="upgradeSacred(3)">
            升级（消耗：{{ OmegaNum.pow(16, player.upgrades[3].sub(1)).format() }}圣魂）
          </button>
        </div>
        <div class="upgrade-item">
          <p>升级4（等级{{ player.upgrades[4].format() }}）：经验需求^1/{{ player.upgrades[4].format() }}</p>
          <button @click="upgradeSacred(4)">
              升级（消耗：{{ OmegaNum.pow(16, player.upgrades[4].sub(1)).format() }}圣魂）
          </button>
        </div>
      </div>
        
      <h3 style="margin-top: 30px; color: #ff6b6b;">炼魂^2系统</h3>
      <button 
          @click="refineSouls2"
          :disabled="!canRefine2"
          style="background-color: #ff6b6b; margin-top: 10px;"
      >
         炼魂^2重置（获得{{ soulGain2.format() }}圣^2魂）
      </button><br />
      {{canRefine2 ? '' : '炼魂^2重置需要玩家等级≥1000且最高敌人等级≥1000'}}
      （当前圣^2魂：{{ player.sacredSouls2.format() }}）
      <div class="upgrades" style="margin-top: 20px;">
        <div class="upgrade-item">
          <p>升级1（等级{{ player.upgrades2[1].format() }}）：每秒伤害^{{ player.upgrades2[1].format() }}</p>
          <button @click="upgradeSacred2(1)">
            升级（消耗：{{ OmegaNum.pow(16, player.upgrades2[1].sub(1)).format() }}圣^2魂）
          </button>
        </div>
        <div class="upgrade-item">
          <p>升级2（等级{{ player.upgrades2[2].format() }}）：获得经验^{{ player.upgrades2[2].format() }}</p>
          <button @click="upgradeSacred2(2)">
            升级（消耗：{{ OmegaNum.pow(16, player.upgrades2[2].sub(1)).format() }}圣^2魂）
          </button>
        </div>
        <div class="upgrade-item">
          <p>升级3（等级{{ player.upgrades2[3].format() }}）：敌人生命^1/{{ player.upgrades2[3].format() }}</p>
          <button @click="upgradeSacred2(3)">
            升级（消耗：{{ OmegaNum.pow(16, player.upgrades2[3].sub(1)).format() }}圣^2魂）
           </button>
        </div>
        <div class="upgrade-item">
          <p>升级4（等级{{ player.upgrades2[4].format() }}）：经验需求^1/{{ player.upgrades2[4].format() }}</p>
          <button @click="upgradeSacred2(4)">
            升级（消耗：{{ OmegaNum.pow(16, player.upgrades2[4].sub(1)).format() }}圣^2魂）
          </button>
        </div>
      </div>
      
      <!-- 炼魂^3系统 -->
      <h3 style="margin-top: 30px; color: #4ecdc4;">炼魂^3系统</h3>
      <button 
          @click="refineSouls3"
          :disabled="!canRefine3"
          style="background-color: #4ecdc4; margin-top: 10px;"
      >
         炼魂^3重置（获得{{ soulGain3.format() }}圣^3魂）
      </button><br />
      {{canRefine3 ? '' : '炼魂^3重置需要玩家等级≥1e7且最高敌人等级≥1e7'}}
      （当前圣^3魂：{{ player.sacredSouls3.format() }}）
      <div class="upgrades" style="margin-top: 20px;">
        <div class="upgrade-item">
          <p>升级1（等级{{ player.upgrades3[1].format() }}）：每秒伤害^{{ player.upgrades3[1].format() }}</p>
          <button @click="upgradeSacred3(1)">
            升级（消耗：{{ OmegaNum.pow(16, player.upgrades3[1].sub(1)).format() }}圣^3魂）
          </button>
        </div>
        <div class="upgrade-item">
          <p>升级2（等级{{ player.upgrades3[2].format() }}）：获得经验^{{ player.upgrades3[2].format() }}</p>
          <button @click="upgradeSacred3(2)">
            升级（消耗：{{ OmegaNum.pow(16, player.upgrades3[2].sub(1)).format() }}圣^3魂）
          </button>
        </div>
        <div class="upgrade-item">
          <p>升级3（等级{{ player.upgrades3[3].format() }}）：敌人生命^1/{{ player.upgrades3[3].format() }}</p>
          <button @click="upgradeSacred3(3)">
            升级（消耗：{{ OmegaNum.pow(16, player.upgrades3[3].sub(1)).format() }}圣^3魂）
           </button>
        </div>
        <div class="upgrade-item">
          <p>升级4（等级{{ player.upgrades3[4].format() }}）：经验需求^1/{{ player.upgrades3[4].format() }}</p>
          <button @click="upgradeSacred3(4)">
            升级（消耗：{{ OmegaNum.pow(16, player.upgrades3[4].sub(1)).format() }}圣^3魂）
          </button>
        </div>
      </div>
      
      <!-- 炼魂^4系统 -->
      <h3 style="margin-top: 30px; color: #96ceb4;">炼魂^4系统</h3>
      <button 
          @click="refineSouls4"
          :disabled="!canRefine4"
          style="background-color: #96ceb4; margin-top: 10px;"
      >
         炼魂^4重置（获得{{ soulGain4.format() }}圣^4魂）
      </button><br />
      {{canRefine4 ? '' : '炼魂^4重置需要玩家等级≥1e13且最高敌人等级≥1e13'}}
      （当前圣^4魂：{{ player.sacredSouls4.format() }}）
      <div class="upgrades" style="margin-top: 20px;">
        <div class="upgrade-item">
          <p>升级1（等级{{ player.upgrades4[1].format() }}）：每秒伤害^{{ player.upgrades4[1].format() }}</p>
          <button @click="upgradeSacred4(1)">
            升级（消耗：{{ OmegaNum.pow(16, player.upgrades4[1].sub(1)).format() }}圣^4魂）
          </button>
        </div>
        <div class="upgrade-item">
          <p>升级2（等级{{ player.upgrades4[2].format() }}）：获得经验^{{ player.upgrades4[2].format() }}</p>
          <button @click="upgradeSacred4(2)">
            升级（消耗：{{ OmegaNum.pow(16, player.upgrades4[2].sub(1)).format() }}圣^4魂）
          </button>
        </div>
        <div class="upgrade-item">
          <p>升级3（等级{{ player.upgrades4[3].format() }}）：敌人生命^1/{{ player.upgrades4[3].format() }}</p>
          <button @click="upgradeSacred4(3)">
            升级（消耗：{{ OmegaNum.pow(16, player.upgrades4[3].sub(1)).format() }}圣^4魂）
           </button>
        </div>
        <div class="upgrade-item">
          <p>升级4（等级{{ player.upgrades4[4].format() }}）：经验需求^1/{{ player.upgrades4[4].format() }}</p>
          <button @click="upgradeSacred4(4)">
            升级（消耗：{{ OmegaNum.pow(16, player.upgrades4[4].sub(1)).format() }}圣^4魂）
          </button>
        </div>
      </div>
    </div>

    <!-- 自动选项卡 -->
    <div v-if="currentTab === 'auto'" class="auto-tab">
 <!-- 自动购买升级 -->
      <div class="auto-section">
        <h3 style="margin-top: 30px; color: #42b883;">自动购买升级</h3>
        <div class="auto-item">
          <label class="auto-checkbox">
            <input type="checkbox" v-model="player.autoBuySacred" :disabled="!canAutoBuySacred">
            自动购买圣魂升级
          </label>
          <span v-if="!canAutoBuySacred" class="lock-info">需要1e6圣魂解锁</span>
        </div>
        <div class="auto-item">
          <label class="auto-checkbox">
            <input type="checkbox" v-model="player.autoBuySacred2" :disabled="!canAutoBuySacred2">
            自动购买圣^2魂升级
          </label>
          <span v-if="!canAutoBuySacred2" class="lock-info">需要1e6圣^2魂解锁</span>
        </div>
        <div class="auto-item">
          <label class="auto-checkbox">
            <input type="checkbox" v-model="player.autoBuySacred3" :disabled="!canAutoBuySacred3">
            自动购买圣^3魂升级
          </label>
          <span v-if="!canAutoBuySacred3" class="lock-info">需要1e6圣^3魂解锁</span>
        </div>
        <div class="auto-item">
          <label class="auto-checkbox">
            <input type="checkbox" v-model="player.autoBuySacred4" :disabled="!canAutoBuySacred4">
            自动购买圣^4魂升级
          </label>
          <span v-if="!canAutoBuySacred4" class="lock-info">需要1e6圣^4魂解锁</span>
        </div>
      </div>
      
      <!-- 自动获得资源 -->
      <div class="auto-section" style="margin-top: 30px;">
        <h3 style="margin-top: 30px; color: #42b883;">自动获得资源</h3>
        <div class="auto-item">
          <label class="auto-checkbox">
            <input type="checkbox" v-model="player.autoGetSacred" :disabled="!canAutoGetSacred">
            每秒自动获得圣魂
          </label>
          <span v-if="!canAutoGetSacred" class="lock-info">需要1e12圣魂解锁</span>
        </div>
        <div class="auto-item">
          <label class="auto-checkbox">
            <input type="checkbox" v-model="player.autoGetSacred2" :disabled="!canAutoGetSacred2">
            每秒自动获得圣^2魂
          </label>
          <span v-if="!canAutoGetSacred2" class="lock-info">需要1e12圣^2魂解锁</span>
        </div>
        <div class="auto-item">
          <label class="auto-checkbox">
            <input type="checkbox" v-model="player.autoGetSacred3" :disabled="!canAutoGetSacred3">
            每秒自动获得圣^3魂
          </label>
          <span v-if="!canAutoGetSacred3" class="lock-info">需要1e12圣^3魂解锁</span>
        </div>
        <div class="auto-item">
          <label class="auto-checkbox">
            <input type="checkbox" v-model="player.autoGetSacred4" :disabled="!canAutoGetSacred4">
            每秒自动获得圣^4魂
          </label>
          <span v-if="!canAutoGetSacred4" class="lock-info">需要1e12圣^4魂解锁</span>
        </div>
      </div>
      
      <!-- 战斗设置 -->
      <div class="auto-section" style="margin-top: 30px;">
        <h3 style="margin-top: 30px; color: #42b883;">战斗设置</h3>
        <div class="auto-item">
          <label class="auto-checkbox">
            <input type="checkbox" v-model="player.isContinuous" :disabled="!canUnlockContinuous">
            持续战斗
          </label>
          <span v-if="!canUnlockContinuous" class="lock-info">需要玩家等级≥5解锁</span>
        </div>
        <div class="auto-item">
          <label class="auto-checkbox">
            <input type="checkbox" v-model="player.isAutoProgress" :disabled="!canUnlockAutoProgress">
            自动推关
          </label>
          <span v-if="!canUnlockAutoProgress" class="lock-info">需要玩家等级≥25解锁</span>
        </div>
      </div>
    </div>
    
    <div v-if="currentTab === 'game'">
      <!-- 玩家信息 -->
      <div class="player-info">
      <h3>玩家状态</h3>
      <p>等级: {{ player.level.formatI() }}</p>
      <p>每秒伤害: {{ totalDamage.format() }}</p>
      <p>经验: {{ player.experience.format() }} / {{ totalExpReq.format() }}</p>
      <div class="exp-bar">
        <div 
          class="exp-bar-fill"
          :style="{ width: expPercentage + '%' }"
        ></div>
        </div>
      </div>
    
      <!-- 敌人选择 -->
      <div class="enemy-selection">
      <h3>选择敌人等级</h3>
      <p>当前选择: {{ enemyLevel }} 级</p>
      <p v-if="skipLevelsInfo" style="color: #42b883; font-weight: bold;">{{ skipLevelsInfo }}</p>
      <div class="level-controls">
        <button 
          @click="selectEnemyLevel(enemyLevel.sub(1).max(1))"
          class="level-button"
        >
          等级-1
        </button>
        <input
          type="range"
          min="1"
          :max="maxSelectableEnemyLevel"
          :value="enemyLevel"
          @input="selectEnemyLevel($event.target.valueAsNumber)"
          class="level-slider"
        >
        <button 
          @click="selectEnemyLevel(enemyLevel.add(1).min(maxSelectableEnemyLevel))"
          class="level-button"
        >
          等级+1
        </button>
      </div>
      </div>
    
      <!-- 战斗界面 -->
      <div class="battle" v-if="isFighting">
      <h3>战斗中...{{ isPaused ? '（已暂停）' : '' }}</h3>
      <p>等级: {{ enemyLevel.formatI() }}</p>
      <p>掉落经验: {{ totalExpGain.format() }}</p>
      <p>生命: {{ enemyCurrentLife.format() }} / {{ enemyMaxLife.format() }}</p>
      <div class="life-bar">
        <div 
          class="life-bar-fill"
          :style="{ width: lifeBarWidth }"
        ></div>
      </div>
      <button @click="toggleFight">
        {{ isPaused ? '继续' : '暂停' }}
      </button>
    </div>
    
    <!-- 统计 -->
    <div class="history">
      <h3>统计</h3>
      <p>最高敌人等级: {{ player.maxDefeatedLevel.formatI() }}</p>
      <p>每秒获得经验: {{ expPerSecond.format() }}</p>
    </div>
    </div>

    <!-- 存档界面 -->
    <div v-if="currentTab === 'save'" class="save-interface">
      四位一体(quaternity) v0.2 作者：6左爷6(dlsdl)
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
        <label class="save-button">
          导入存档<input type="file" accept=".json" @change="importSave" hidden>
        </label>
        <button @click="resetGameData" class="save-button">
          重置游戏数据
        </button>
      </div>
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
  padding: 10px;
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
}

.exp-bar {
  width: 100%;
  height: 20px;
  background-color: #333;
  border-radius: 10px;
  overflow: hidden;
  margin: 10px 0;
}

.exp-bar-fill {
  height: 100%;
  background-color: #646cff;
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

.level-button {
  padding: 5px 10px;
  background-color: #646cff;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.level-button:hover {
  background-color: #535bf2;
}

.checkbox-group {
  display: flex;
  gap: 20px;
  margin-top: 10px;
}

.continuous-checkbox {
  display: flex;
  align-items: center;
  gap: 5px;
}

/* 自动选项卡样式 */
.auto-tab {
  padding: 10px;
}

.auto-section {
  background-color: rgba(255, 255, 255, 0.1);
  border-radius: 10px;
  padding: 10px;
  margin-bottom: 10px;
}

.auto-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin: 10px 0;
}

.auto-checkbox {
  display: flex;
  align-items: center;
  gap: 5px;
}

.lock-info {
  color: #ff6b6b;
  font-size: 14px;
}

p {
  margin: 8px 0;
}
</style>
