// 安全解包响应式变量
const lifeBarWidth = computed(() => {
  const current = enemyCurrentLife.value?.toNumber() || 0;
  const max = enemyMaxLife.value?.toNumber() || 1;
  return (current / max * 100) + '%';
});

// 修复存档数据访问
const legacyFormat = !importedData.player;
player.maxDefeatedLevel = new OmegaNum(
  legacyFormat ? importedData.maxDefeatedLevel : 
  (importedData.player?.maxDefeatedLevel || 1)
);

// 升级项安全访问
const totalDamage = computed(() =>
  player.level.value.mul(
    player.upgrades[1].value.gt(0) ?
    player.upgrades[1].value :
    new OmegaNum(1)
  )
);