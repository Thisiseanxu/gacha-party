<template>
  <FloatingHomeButton />
  <div class="background">
    <div v-if="viewState === 'input'" class="gacha-analysis-container">
      <div v-if="viewState === 'input'" class="input-section">
        <h2 class="input-title">抽卡记录分析</h2>
        <p>此页面可分析使用抽卡记录导出工具导出的抽卡数据<br />
          出于安全原因，请加 <a class="highlight"
            href="https://qm.qq.com/cgi-bin/qm/qr?k=PD3VWuDfxO_hAVZQBreK1CjvWORTkNN2&jump_from=webapi&authKey=c4Sos3R4opf3VqerCwpPX+IOmwZUDm4hqkyT7qDGhta2fAhdUETlxFZ9wDrcRu1z"
            target="_blank">
            Q群1049576192</a> 获取记录
        </p>
        <p class="input-description">请在下方文本框粘贴您的抽卡记录 JSON 数据，或上传导出的文件。</p>

        <textarea v-model="jsonInput" id="jsonInput" class="json-textarea"
          placeholder='请在此处粘贴 JSON 数据... 例如：{"version":2,"9999999":{"9":[{"id":7579416,"gacha_id":9,"item_id":"151406","created_at":1751324096},...]}}'></textarea>

        <div class="button-group">
          <button @click="handleJsonAnalysis" class="action-button">开始分析</button>
          <label class="file-upload-button action-button">
            上传文件
            <input type="file" @change="handleFileUpload" accept=".json,application/json" style="display: none;" />
          </label>
        </div>

        <p v-if="errorMessage" class="error-message">{{ errorMessage }}</p>
        <p class="input-description">本网页完全开源，可查看<a class="highlight" href="https://github.com/Thisiseanxu/gacha-party"
            target="_blank">Github链接</a>提出意见/贡献代码。</p>
      </div>
    </div>

    <div v-if="viewState === 'analysis'" class="gacha-analysis-container">

      <div v-if="viewState === 'analysis'" class="gacha-analysis-page">
        <button @click="resetView" class="button">← 分析新文件</button>

        <div v-if="analysis && analysis.totalPulls > 0">
          <div class="header">
            <SelectorComponent v-model="CurrentSelectedPool" :options="cardPoolOptions" option-text-key="name"
              option-value-key="id">
              <template #trigger>
                <div class="title-bar">
                  <span>
                    {{ playerId }}-{{ CARDPOOLS_NAME_MAP[CurrentSelectedPool] }}
                    {{ CurrentSelectedPool !== 'Limited' ? '(计算垫抽)' : '' }}
                  </span>
                </div>
              </template>
            </SelectorComponent>
            <div :class="{ 'total-pulls': true, 'highlight': CurrentSelectedPool !== 'Limited' }">{{
              urAnalysis.totalPulls
            }} <span class="pulls-text">抽</span>
            </div>

            <span v-if="urAnalysis.SinglePulls > 0" class="single-pulls-text">{{ '此卡池共计' + urAnalysis.SinglePulls + '抽'
              }}
            </span>
            <div class="pity-counters">
              <div class="pity-item">
                <span>距上个限定 </span>
                <span class="pity-count SP">{{ analysis.SP }}</span>
              </div>
              <div class="pity-item">
                <span>距上个SSR</span>
                <span class="pity-count SSR">{{ analysis.SSR }}</span>
              </div>
            </div>
            <div class="date-range">{{ analysis.dateRange }}</div>
          </div>

          <div class="stats-overview">
            <div class="stat-box">
              <div>限定平均抽数</div>
              <div v-if="urAnalysis.avgPullsForSP > 0"
                :class="{ 'stat-value': true, 'highlight': CurrentSelectedPool !== 'Limited' }">{{
                  urAnalysis.avgPullsForSP.toFixed(2) }} 抽
              </div>
              <div v-else class="stat-value">暂无数据</div>
            </div>
            <div class="stat-box">
              <div>SSR平均抽数</div>
              <div v-if="urAnalysis.avgPullsForSSR > 0" class="stat-value">{{
                urAnalysis.avgPullsForSSR.toFixed(2)
                }} 抽
              </div>
              <div v-else-if="CurrentSelectedPool !== 'Limit'" class="stat-value">单池无法统计</div>
              <div v-else class="stat-value">暂无数据</div>
            </div>
            <div class="stat-box">
              <div>最非限定</div>
              <div v-if="urAnalysis.maxSP > 0"
                :class="{ 'stat-value': true, 'highlight': CurrentSelectedPool !== 'Limited' }">{{
                  urAnalysis.maxSP }} 抽
              </div>
              <div v-else class="stat-value">暂无数据</div>
            </div>
            <div class="stat-box">
              <div>最欧限定</div>
              <div v-if="urAnalysis.minSP > 0"
                :class="{ 'stat-value': true, 'highlight': CurrentSelectedPool !== 'Limited' }">{{
                  urAnalysis.minSP }} 抽
              </div>
              <div v-else class="stat-value">暂无数据</div>
            </div>
          </div>

          <div class="history-list" ref="historyListRef">
            <div v-for="(item, index) in urAnalysis.SPHistory" :key="index" class="history-item"
              :style="getHistoryItemStyle(item)">
              <div class="char-info">
                <img :src="item.imageUrl" :alt="item.name" class="char-avatar">
                <span class="char-name">{{ item.name }}</span>
              </div>
              <div class="pull-info">
                <span class="pull-count">{{ item.count }}</span>
              </div>
            </div>
          </div>

          <div class="full-history-section">
            <h3 class="section-title">{{ CARDPOOLS_NAME_MAP[CurrentSelectedPool] }} - {{ CurrentSelectedPool
              ===
              "Limit" ? '完整' : '卡池' }}抽卡历史</h3>
            <div class="full-history-list">
              <div v-for="item in paginatedHistory" :key="item.gacha_id" :class="['full-history-item', item.rarity]">
                <div class="char-info">
                  <img :src="item.imageUrl" :alt="item.name" class="char-avatar">
                  <span class="char-name">{{ item.name }}</span>
                </div>
                <span :class="['rarity-' + item.rarity]">{{ item.date }}</span>
              </div>
              <p v-if="fullHistory.length === 0" class="no-history-text">暂无抽卡历史。</p>
            </div>
            <div v-if="totalPages > 1" class="pagination-controls">
              <button @click="prevPage" :disabled="currentPage === 1">上一页</button>
              <span>
                第
                <input type="number" id="LimitPageInput" class="page-input" v-model="pageInput" @keyup.enter="goToPage"
                  @blur="goToPage" min="1" :max="totalPages" />
                页 / 共 {{ totalPages }} 页
              </span>
              <button @click="nextPage" :disabled="currentPage === totalPages">下一页</button>
            </div>
          </div>
          <div style="text-align: center; padding: 20px 0;">
            <button @click="exportLimitData" class="button">导出{{ CARDPOOLS_NAME_MAP[CurrentSelectedPool]
              }}卡池记录</button>
          </div>
        </div>

        <p v-if="(!analysis || analysis.totalPulls === 0)">
          欸？好像没有抽卡记录
        </p>

      </div>
    </div>

    <div v-if="viewState === 'analysis'" class="gacha-analysis-container">

      <div v-if="viewState === 'analysis'" class="gacha-analysis-page">
        <button @click="resetView" class="button">← 分析新文件</button>

        <div v-if="normalAnalysis && normalAnalysis.totalPulls > 0" class="permanent-pool-section">
          <div class="header">
            <div class="title-bar">
              <span>{{ playerId }}-{{ CARDPOOLS_NAME_MAP["Normal"] }}</span>
            </div>
            <div class="total-pulls">{{ normalAnalysis.totalPulls }} <span class="pulls-text">抽</span></div>
            <div class="pity-counters">
              <div class="pity-item">
                <span>距上个SSR</span>
                <span class="pity-count SSR">{{ normalAnalysis.SSR }}</span>
              </div>
            </div>
            <div class="date-range">{{ normalAnalysis.dateRange }}</div>
          </div>

          <div class="stats-overview">
            <div class="stat-box">
              <div>已获取SSR数量</div>
              <div class="stat-value">{{ normalAnalysis.totalSSRs }}</div>
            </div>
            <div class="stat-box">
              <div>SSR平均抽数</div>
              <div v-if="normalAnalysis.avgPullsForSSR > 0" class="stat-value">{{
                normalAnalysis.avgPullsForSSR.toFixed(2) }} 抽</div>
              <div v-else class="stat-value">暂无数据</div>
            </div>
            <div class="stat-box">
              <div>最非SSR</div>
              <div v-if="normalAnalysis.maxSSR > 0" class="stat-value">{{ normalAnalysis.maxSSR }} 抽</div>
              <div v-else class="stat-value">暂无数据</div>
            </div>
            <div class="stat-box">
              <div>最欧SSR</div>
              <div v-if="normalAnalysis.minSSR > 0" class="stat-value">{{ normalAnalysis.minSSR }} 抽</div>
              <div v-else class="stat-value">暂无数据</div>
            </div>
          </div>

          <div class="history-list" ref="normalHistoryListRef">
            <div v-for="(item, index) in normalAnalysis.SSRHistory" :key="index" class="history-item"
              :style="getHistoryItemStyle(item, true)">
              <div class="char-info">
                <img :src="item.imageUrl" :alt="item.name" class="char-avatar">
                <span class="char-name">{{ item.name }}</span>
              </div>
              <div class="pull-info">
                <span class="pull-count">{{ item.count }}</span>
              </div>
            </div>
          </div>

          <div class="full-history-section">
            <h3 class="section-title">{{ CARDPOOLS_NAME_MAP["Normal"] }} - 完整抽卡历史</h3>
            <div class="full-history-list">
              <div v-for="item in normalPaginatedHistory" :key="item.gacha_id"
                :class="['full-history-item', item.rarity]">
                <div class="char-info">
                  <img :src="item.imageUrl" :alt="item.name" class="char-avatar">
                  <span class="char-name">{{ item.name }}</span>
                </div>
                <span :class="['rarity-' + item.rarity]">{{ item.date }}</span>
              </div>
              <p v-if="normalFullHistory.length === 0" class="no-history-text">暂无抽卡历史。</p>
            </div>
            <div v-if="normalTotalPages > 1" class="pagination-controls">
              <button @click="prevNormalPage" :disabled="normalCurrentPage === 1">上一页</button>
              <span>
                第
                <input type="number" id="NormalPageInput" class="page-input" v-model="normalPageInput"
                  @keyup.enter="goToNormalPage" @blur="goToNormalPage" min="1" :max="normalTotalPages" />
                页 / 共 {{ normalTotalPages }} 页
              </span>
              <button @click="nextNormalPage" :disabled="normalCurrentPage === normalTotalPages">下一页</button>
            </div>
          </div>
          <div style="text-align: center; padding: 20px 0;">
            <button @click="exportNormalData" class="button">导出{{ CARDPOOLS_NAME_MAP['Normal']
              }}卡池记录</button>
          </div>
        </div>

        <p v-if="(!analysis || analysis.totalPulls === 0) && (!normalAnalysis || normalAnalysis.totalPulls === 0)">
          欸？好像没有抽卡记录
        </p>

      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue';
import { cardMap } from '@/data/cards.js';
import * as RARITY from '@/data/rarity.js';
import { colors } from '@/styles/colors.js';
import { logger } from '@/utils/logger.js';
import SelectorComponent from '@/components/SelectorComponent.vue';
import pako from 'pako';
import FloatingHomeButton from '@/components/FloatingHomeButton.vue';

const CARDPOOLS_NAME_MAP = {
  'Normal': '常驻扭蛋',
  'Limited': '限定扭蛋',
  '9': '常驻扭蛋',
  '29': '车手盲盒机',
  '40': '塔菲扭蛋',
  '41': '童话国盲盒机',
  '42': '扭蛋大作战',
};

const LIMITED_CARD_POOLS_ID = ['29', '40', '41', '42']; // 限定卡池ID列表

const viewState = ref('input'); // 'input' 则为用户输入 'analysis' 则为用户上传json文件
const jsonInput = ref(''); // 存储用户输入的 JSON 数据
const playerId = ref(''); // 存储玩家ID
const LimitGachaData = ref([]); // 存储限定卡池抽卡记录
const NormalGachaData = ref([]); // 存储常驻卡池抽卡记录
const CurrentSelectedPool = ref("Limited"); // 控制限定卡池筛选指定卡池的抽卡记录
const errorMessage = ref('');

// CurrentSelectedPool.value = 29; // DEBUG：模拟用户选择了某个卡池

const cardPoolOptions = ref([
  { id: 'Limited', name: CARDPOOLS_NAME_MAP['Limited'] },
  ...LIMITED_CARD_POOLS_ID.map(id => ({ id, name: CARDPOOLS_NAME_MAP[id] }))
]);

const getCardInfoAndRemovePrefix = (itemId) => {
  // id格式为15xxxx，而cardMap中没有15前缀，直接是xxxx，因此需要转换
  let cardId = itemId;
  if (itemId.startsWith('15')) {
    cardId = itemId.slice(2); // 去掉前缀 "15"
  }
  return cardMap.get(cardId) || null;
};

const handleJsonAnalysis = () => {
  errorMessage.value = '';

  if (!jsonInput.value.trim()) {
    errorMessage.value = '请输入JSON数据！';
    return;
  }

  let finalData;
  try {
    let parsedData = JSON.parse(jsonInput.value);

    // 检查是否是压缩格式
    if (parsedData && parsedData.compressed === true && typeof parsedData.data === 'string') {
      try {
        // Base64 解码
        const binaryString = atob(parsedData.data);
        const len = binaryString.length;
        const bytes = new Uint8Array(len);
        for (let i = 0; i < len; i++) {
          bytes[i] = binaryString.charCodeAt(i);
        }

        // Gzip 解压
        const decompressedString = pako.inflate(bytes, { to: 'string' });
        finalData = JSON.parse(decompressedString);

      } catch (e) {
        errorMessage.value = `解压或解析压缩数据时失败，请确认数据是否正确。错误: ${e.message}`;
        return;
      }
    } else {
      // 如果不是压缩格式，直接使用
      finalData = parsedData;
    }

  } catch (error) {
    errorMessage.value = `JSON 格式无法解析，请检查。错误详情: ${error.message}`;
    return;
  }

  // --- 后续逻辑使用 finalData ---

  if (typeof finalData !== 'object' || finalData === null || Array.isArray(finalData)) {
    errorMessage.value = '数据格式错误：顶层结构必须是一个对象 ( 形如 {"key": "value", ...} )。';
    return;
  }

  if (typeof finalData.version !== 'number' || finalData.version < 2) {
    errorMessage.value = '您的数据版本不正确。请确保使用最新版盲盒派对抽卡记录导出工具导出数据！';
    return;
  }

  playerId.value = Object.keys(finalData).find(key => key !== 'version');
  if (!playerId.value) {
    errorMessage.value = '数据格式错误：找不到玩家ID！';
    return;
  }

  const playerData = finalData[playerId.value];
  if (typeof playerData !== 'object' || playerData === null || Object.keys(playerData).length === 0) {
    errorMessage.value = '数据格式错误：玩家ID下没有任何卡池对象！';
    return;
  }

  const gachaPools = Object.values(playerData);
  if (!gachaPools.some(pool => Array.isArray(pool))) {
    errorMessage.value = '数据格式错误：未找到有效的卡池数据！';
    return;
  }

  const LimitGachaRecords = [];
  const NormalGachaRecords = [];
  for (const [gachaId, records] of Object.entries(playerData)) {
    if (gachaId === '9') {
      NormalGachaRecords.push(...records);
    } else if (LIMITED_CARD_POOLS_ID.includes(gachaId)) {
      LimitGachaRecords.push(...records);
    }
  }

  for (const item of LimitGachaRecords) {
    if (typeof item !== 'object' || item === null || !('id' in item) || !('item_id' in item)) {
      errorMessage.value = '数据格式错误：限定卡池抽卡记录缺少 "id" 或 "item_id" 字段';
      return;
    }
  }

  for (const item of NormalGachaRecords) {
    if (typeof item !== 'object' || item === null || !('id' in item) || !('item_id' in item)) {
      errorMessage.value = '数据格式错误：常驻卡池抽卡记录缺少 "id" 或 "item_id" 字段';
      return;
    }
  }

  LimitGachaData.value = LimitGachaRecords;
  NormalGachaData.value = NormalGachaRecords;
  viewState.value = 'analysis';
};

// 处理文件上传
const handleFileUpload = (event) => {
  const file = event.target.files[0];
  if (!file) return;

  const reader = new FileReader();
  reader.onload = (e) => {
    jsonInput.value = e.target.result; // 这里为了方便处理，直接把文件内容放进文本框，这样不用做两个逻辑，我真是个天才
    handleJsonAnalysis();
  };
  reader.onerror = () => {
    errorMessage.value = '读取文件时发生错误。';
  };
  reader.readAsText(file);
  // 清空事件
  event.target.value = '';
};

// 重置网页获取其他输入
const resetView = () => {
  viewState.value = 'input';
  jsonInput.value = '';
  LimitGachaData.value = [];
  NormalGachaData.value = [];
  errorMessage.value = '';
};

// 计算列表平均值的通用函数
const calculateAverage = (arr) => arr.length > 0 ? (arr.reduce((a, b) => a + b, 0) / arr.length) : 0;

// 限定卡池分析逻辑
const analysis = computed(() => {
  // 仅当有有效数据时才执行计算
  if (LimitGachaData.value.length === 0) return null;

  // 将数据改成从最久远到最近排序，方便计算抽数
  const records = [...LimitGachaData.value].sort((a, b) => a.created_at - b.created_at || a.id - b.id);

  let SPCounter = 0;
  let SSRCounter = 0;

  const SPHistory = [];
  const SSRHistory = [];


  records.forEach((record) => {
    const cardInfo = getCardInfoAndRemovePrefix(record.item_id);
    if (!cardInfo) {
      logger.warn(`未找到 item_id: ${record.item_id} 的信息，已跳过。`);
      return;
    }

    SPCounter++;
    SSRCounter++;

    if (cardInfo.rarity === RARITY.SP) {
      SPHistory.unshift({
        ...cardInfo,
        count: SPCounter,
        gacha_id: record.gacha_id,
      });
      SPCounter = 0;
    }

    if (cardInfo.rarity === RARITY.SSR) {
      SSRHistory.push({ ...cardInfo, count: SSRCounter, gacha_id: record.gacha_id });
      SSRCounter = 0;
    }
  });

  const totalPulls = records.length;
  const formatDate = (timestamp) => {
    if (!timestamp) return '';
    const date = new Date(timestamp * 1000);
    return `${date.getFullYear()}.${String(date.getMonth() + 1).padStart(2, '0')}.${String(date.getDate()).padStart(2, '0')}`;
  };
  const startDate = formatDate(records[0]?.created_at);
  const endDate = formatDate(records[records.length - 1]?.created_at);

  return {
    totalPulls,
    SinglePulls: 0,
    SP: SPCounter,
    SSR: SSRCounter,
    dateRange: `${startDate} - ${endDate}`,
    avgPullsForSP: calculateAverage(SPHistory.map(item => item.count)),
    avgPullsForSSR: calculateAverage(SSRHistory.map(item => item.count)),
    maxSP: Math.max(...SPHistory.map(item => item.count), 0),
    minSP: Math.min(...SPHistory.map(item => item.count), Infinity),
    SPHistory: SPHistory
  };
});

// 限定卡池单卡池分析逻辑
const urAnalysis = computed(() => {
  if (!analysis.value) return null;
  if (CurrentSelectedPool.value !== 'Limited') {
    // 如果选择了特定卡池，则只分析该卡池的记录，注意转换成数字
    const filteredHistory = analysis.value.SPHistory.filter(item => item.gacha_id === Number(CurrentSelectedPool.value));
    if (filteredHistory.length === 0) {
      return {
        totalPulls: 0,
        SinglePulls: 0,
        avgPullsForSP: 0,
        avgPullsForSSR: 0,
        maxSP: 0,
        minSP: 0,
        SPHistory: [],
      };
    }
    return {
      totalPulls: filteredHistory.reduce((sum, item) => sum + item.count, 0),
      SinglePulls: fullHistory.value.length,
      avgPullsForSP: calculateAverage(filteredHistory.map(item => item.count)),
      avgPullsForSSR: 0,
      maxSP: Math.max(...filteredHistory.map(item => item.count), 0),
      minSP: Math.min(...filteredHistory.map(item => item.count), Infinity),
      SPHistory: filteredHistory
    };
  }
  return { // 如果没有选择特定卡池，则返回全部限定卡池的分析数据
    totalPulls: analysis.value.totalPulls,
    SinglePulls: analysis.value.SinglePulls,
    avgPullsForSP: analysis.value.avgPullsForSP,
    avgPullsForSSR: analysis.value.avgPullsForSSR,
    maxSP: analysis.value.maxSP,
    minSP: analysis.value.minSP,
    SPHistory: analysis.value.SPHistory,
  };
});

// 常驻卡池分析逻辑
const normalAnalysis = computed(() => {
  if (NormalGachaData.value.length === 0) return null;

  const records = [...NormalGachaData.value].sort((a, b) => a.created_at - b.created_at || a.id - b.id);

  let SSRCounter = 0;
  const SSRHistory = [];
  const SSRPulls = [];

  records.forEach((record) => {
    const cardInfo = getCardInfoAndRemovePrefix(record.item_id);
    if (!cardInfo) {
      logger.warn(`(常驻池)未找到 item_id: ${record.item_id} 的信息，已跳过。`);
      return;
    }

    SSRCounter++;

    if (cardInfo.rarity === RARITY.SSR) {
      SSRHistory.unshift({
        ...cardInfo,
        count: SSRCounter,
      });
      SSRPulls.push(SSRCounter);
      SSRCounter = 0;
    }
  });

  const totalPulls = records.length;
  const formatDate = (timestamp) => {
    if (!timestamp) return '';
    const date = new Date(timestamp * 1000);
    return `${date.getFullYear()}.${String(date.getMonth() + 1).padStart(2, '0')}.${String(date.getDate()).padStart(2, '0')}`;
  };
  const startDate = formatDate(records[0]?.created_at);
  const endDate = formatDate(records[records.length - 1]?.created_at);

  return {
    totalPulls,
    SSR: SSRCounter,
    dateRange: `${startDate} - ${endDate}`,
    avgPullsForSSR: calculateAverage(SSRPulls),
    maxSSR: SSRPulls.length > 0 ? Math.max(...SSRPulls) : 0,
    minSSR: SSRPulls.length > 0 ? Math.min(...SSRPulls) : 0,
    SSRHistory: SSRHistory,
    totalSSRs: SSRPulls.length,
  };
});

/**
 * 根据抽数计算背景样式
 * @param {object} item - 包含count属性的历史记录项
 * @param {boolean} isNormal - 是否为常驻池SSR（常驻池没有SP，保底阈值不同）
 * @returns {object} - 一个包含背景样式的对象
 */
const getHistoryItemStyle = (item, isNormal = false) => {
  const maxCount = 60;
  const count = item.count;
  const percentage = (count / maxCount) * 100;

  let progressBarColor;

  // 根据不同卡池和抽数应用不同颜色
  if ((isNormal && count < 9) || (!isNormal && count < 30)) {
    progressBarColor = colors.progressBar.low;
  } else if ((isNormal && count < 19) || (!isNormal && count < 41)) {
    progressBarColor = colors.progressBar.medium;
  } else {
    progressBarColor = colors.progressBar.high;
  }

  const backgroundColor = colors.progressBar.background;
  return {
    background: `linear-gradient(to right, ${progressBarColor} ${percentage}%, ${backgroundColor} ${percentage}%)`
  };
};

// 限定卡池分页逻辑
const currentPage = ref(1);
const itemsPerPage = ref(10);
const pageInput = ref(1);

const fullHistory = computed(() => {
  if (LimitGachaData.value.length === 0) return [];
  let filteredData = [...LimitGachaData.value]
  if (CurrentSelectedPool.value !== 'Limited') {
    filteredData = filteredData.filter(record => record.gacha_id === Number(CurrentSelectedPool.value));
  }
  return filteredData.sort((a, b) => b.created_at - a.created_at || b.id - a.id).map(record => {
    const cardInfo = getCardInfoAndRemovePrefix(record.item_id);
    const defaultCard = { name: `未知角色 (${record.item_id})`, rarity: RARITY.R, imageUrl: '/images/cards/placeholder.webp' };
    const createdAt = new Date(record.created_at * 1000);
    const formattedDate = `${createdAt.getFullYear().toString().slice(-2)}/${String(createdAt.getMonth() + 1).padStart(2, '0')}/${String(createdAt.getDate()).padStart(2, '0')} ${String(createdAt.getHours()).padStart(2, '0')}:${String(createdAt.getMinutes()).padStart(2, '0')}:${String(createdAt.getSeconds()).padStart(2, '0')}`;
    return { ...(cardInfo || defaultCard), gacha_id: record.id, date: formattedDate };
  });
});

const totalPages = computed(() => Math.ceil(fullHistory.value.length / itemsPerPage.value));
const paginatedHistory = computed(() => {
  const start = (currentPage.value - 1) * itemsPerPage.value;
  const end = start + itemsPerPage.value;
  return fullHistory.value.slice(start, end);
});

const nextPage = () => {
  if (currentPage.value < totalPages.value) currentPage.value++;
};
const prevPage = () => {
  if (currentPage.value > 1) currentPage.value--;
};

// 跳转到指定页面的函数（限定池）
const goToPage = () => {
  const page = Math.floor(Number(pageInput.value));
  if (!isNaN(page) && page >= 1 && page <= totalPages.value) {
    currentPage.value = page;
  } else {
    // 如果输入无效，则将输入框的值重置为当前页码
    pageInput.value = currentPage.value;
  }
};

// 常驻卡池分页逻辑
const normalCurrentPage = ref(1);
// itemsPerPage 可以共用
const normalPageInput = ref(1);

const normalFullHistory = computed(() => {
  if (NormalGachaData.value.length === 0) return [];
  return [...NormalGachaData.value].sort((a, b) => b.created_at - a.created_at || b.id - a.id).map(record => {
    const cardInfo = getCardInfoAndRemovePrefix(record.item_id);
    const defaultCard = { name: `未知角色 (${record.item_id})`, rarity: RARITY.R, imageUrl: '/images/cards/placeholder.webp' };
    const createdAt = new Date(record.created_at * 1000);
    const formattedDate = `${createdAt.getFullYear().toString().slice(-2)}/${String(createdAt.getMonth() + 1).padStart(2, '0')}/${String(createdAt.getDate()).padStart(2, '0')} ${String(createdAt.getHours()).padStart(2, '0')}:${String(createdAt.getMinutes()).padStart(2, '0')}:${String(createdAt.getSeconds()).padStart(2, '0')}`;
    return { ...(cardInfo || defaultCard), gacha_id: record.id, date: formattedDate };
  });
});

const normalTotalPages = computed(() => Math.ceil(normalFullHistory.value.length / itemsPerPage.value));
const normalPaginatedHistory = computed(() => {
  const start = (normalCurrentPage.value - 1) * itemsPerPage.value;
  const end = start + itemsPerPage.value;
  return normalFullHistory.value.slice(start, end);
});

const nextNormalPage = () => {
  if (normalCurrentPage.value < normalTotalPages.value) normalCurrentPage.value++;
};
const prevNormalPage = () => {
  if (normalCurrentPage.value > 1) normalCurrentPage.value--;
};

// 跳转到指定页面的函数（常驻池）
const goToNormalPage = () => {
  const page = Math.floor(Number(normalPageInput.value));
  if (!isNaN(page) && page >= 1 && page <= normalTotalPages.value) {
    normalCurrentPage.value = page;
  } else {
    // 如果输入无效，则将输入框的值重置为当前页码
    normalPageInput.value = normalCurrentPage.value;
  }
};

// 监听限定卡池选择变化，重置页码为1
watch(CurrentSelectedPool, () => {
  currentPage.value = 1;
});

// 监听 currentPage 的变化，同步更新输入框的值
watch(currentPage, (newPage) => {
  pageInput.value = newPage;
});

// 监听 normalCurrentPage 的变化，同步更新输入框的值
watch(normalCurrentPage, (newPage) => {
  normalPageInput.value = newPage;
});


// 通用的导出卡池数据函数
const exportToCsv = (filename, historyData) => {
  if (historyData.length === 0) {
    alert('没有数据可供导出。');
    return;
  }

  const headers = ['角色名称', '稀有度', '抽到时间'];
  const rows = historyData.map(item => {
    const { name, rarity, date } = item;
    const rarityText = rarity === 'SP' ? '限定' : rarity;
    const safeName = `"${name}"`;
    return [safeName, rarityText, date];
  });

  const csvContent = [headers.join(','), ...rows.map(row => row.join(','))].join('\n');
  const bom = new Uint8Array([0xEF, 0xBB, 0xBF]);
  const blob = new Blob([bom, csvContent], { type: 'text/csv;charset=utf-8;' });
  const link = document.createElement('a');
  const url = URL.createObjectURL(blob);
  link.setAttribute('href', url);
  link.setAttribute('download', filename);
  link.style.visibility = 'hidden';
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  URL.revokeObjectURL(url);
};

// 限定卡池导出
const exportLimitData = () => {
  exportToCsv(CARDPOOLS_NAME_MAP[CurrentSelectedPool.value] + '抽卡记录.csv', fullHistory.value);
};

// 常驻卡池导出
const exportNormalData = () => {
  exportToCsv('常驻卡池抽卡记录.csv', normalFullHistory.value);
};


// CSS颜色常量
const colorBgPrimary = colors.background.primary;
const colorBgContent = colors.background.content;
const colorBgLight = colors.background.light;
const colorBgLighter = colors.background.lighter;
const colorBgHover = colors.background.hover;
const colorBgAvatar = colors.background.avatar;

const colorTextPrimary = colors.text.primary;
const colorTextSecondary = colors.text.secondary;
const colorTextTertiary = colors.text.tertiary;
const colorTextDark = colors.text.dark;
const colorTextDisabled = colors.text.disabled;
const colorTextLight = colors.text.light;
const colorTextHighlight = colors.text.highlight;

const colorBrandPrimary = colors.brand.primary;
const colorBrandHover = colors.brand.hover;

const colorRarityUr = colors.rarity.ur;
const colorRaritySsr = colors.rarity.ssr;
const colorRaritySr = colors.rarity.sr;
const colorRarityR = colors.rarity.r;

const colorStatusError = colors.status.error;
const colorStatusErrorBg = colors.status.errorBg;

const colorScrollbar = colors.scrollbar;
const colorTextShadow = colors.textShadow;
</script>

<style scoped>
.background {
  padding: 1vh 1vw;
  min-height: 100vh;
  background-color: v-bind(colorBgPrimary);
  color: v-bind(colorTextPrimary);
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  display: flex;
  flex-direction: row;
  align-items: flex-start;
  flex-wrap: wrap;
  justify-content: center;
  gap: 5vw;
}

.gacha-analysis-container {
  background-color: v-bind(colorBgContent);
  padding: 16px;
  margin: 8px;
  min-width: 300px;
  width: 450px;
  border-radius: 12px;
}

/* --- 上传记录区域 --- */
.input-section {
  display: flex;
  flex-direction: column;
  gap: 16px;
  text-align: center;
}

.input-title {
  font-size: 1.8rem;
  margin: 0;
}

.input-description {
  color: v-bind(colorTextSecondary);
  font-size: 0.9rem;
  margin: 0;
}

.json-textarea {
  min-height: 200px;
  background-color: v-bind(colorBgLight);
  border: 1px solid v-bind(colorBorderPrimary);
  /* 假设定义了 colorBorderPrimary */
  border-radius: 8px;
  color: v-bind(colorTextPrimary);
  padding: 12px;
  font-family: 'Courier New', Courier, monospace;
  font-size: 0.85rem;
  resize: vertical;
  width: auto;
}

.json-textarea:focus {
  outline: none;
  border-color: v-bind(colorBrandPrimary);
}

.button-group {
  display: flex;
  gap: 12px;
}

.action-button {
  flex-grow: 1;
  padding: 12px 20px;
  border: none;
  border-radius: 8px;
  background-color: v-bind(colorBrandPrimary);
  color: v-bind(colorTextDark);
  font-size: 1rem;
  font-weight: bold;
  cursor: pointer;
  transition: background-color 0.2s;
}

.action-button:hover {
  background-color: v-bind(colorBrandHover);
}

.file-upload-button {
  text-align: center;
}

.error-message {
  color: v-bind(colorStatusError);
  background-color: v-bind(colorStatusErrorBg);
  border: 1px solid v-bind(colorStatusError);
  padding: 10px;
  border-radius: 8px;
  margin: 0;
  font-size: 0.9rem;
  word-break: break-word;
}

/* --- 分析结果区域 --- */
.gacha-analysis-page>div:not(:first-child) {
  padding-top: 24px;
  margin-top: 24px;
  border-top: 2px solid v-bind(colorBgLight);
}

.button {
  background-color: v-bind(colorBgLighter);
  color: v-bind(colorTextLight);
  border: none;
  padding: 8px 12px;
  border-radius: 6px;
  cursor: pointer;
  margin-bottom: 8px;
  font-weight: bold;
}

.button:hover {
  background-color: v-bind(colorBgHover);
}

.header {
  padding: 10px;
}

.title-bar {
  display: flex;
  justify-content: flex-start;
  font-size: 1.2rem;
  font-weight: bold;
}

.total-pulls {
  font-size: 3.5rem;
  font-weight: bold;
  letter-spacing: -2px;
}

.highlight {
  color: v-bind(colorTextHighlight);
}

.highlight:visited {
  color: v-bind(colorTextHighlight);
}

.pulls-text {
  font-size: 1rem;
  font-weight: normal;
  margin-left: 8px;
}

.pity-counters {
  display: flex;
  gap: 20px;
  margin-top: 5px;
  background-color: v-bind(colorBgLight);
  padding: 10px;
  border-radius: 8px;
}

.pity-item {
  display: flex;
  align-items: center;
  gap: 8px;
  color: v-bind(colorTextSecondary);
}

.pity-count {
  font-weight: bold;
  font-size: 1.2rem;
}

.pity-count.SP {
  color: v-bind(colorRarityUr);
}

.pity-count.SSR {
  color: v-bind(colorRaritySsr);
}

.date-range {
  margin-top: 10px;
  color: v-bind(colorTextTertiary);
  font-size: 0.9rem;
}

/* 统计数据概览 */
.stats-overview {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}

.stat-box {
  background-color: v-bind(colorBgLight);
  padding: 15px;
  border-radius: 8px;
  text-align: center;
}

.stat-box div:first-child {
  color: v-bind(colorTextSecondary);
  font-size: 0.9rem;
}

.stat-box .stat-value {
  margin-top: 8px;
  font-size: 1.2rem;
  font-weight: bold;
}

.history-list {
  margin-top: 20px;
  display: flex;
  flex-direction: column;
  gap: 8px;
  max-height: 600px;
  overflow-y: auto;
  scrollbar-width: thin;
  scrollbar-color: v-bind(colorScrollbar) transparent;
  transition: scrollbar-color 0.5s ease-out;
}

/* --- 适配 Webkit 内核浏览器 (Chrome, Edge, Safari) --- */
.history-list::-webkit-scrollbar {
  width: 6px;
}

.history-list::-webkit-scrollbar-track {
  background: transparent;
}

.history-list::-webkit-scrollbar-thumb {
  background-color: v-bind(colorScrollbar);
  border-radius: 3px;
}

.history-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 12px;
  border-radius: 8px;
  position: relative;
  z-index: 1;
}

.char-info {
  display: flex;
  align-items: center;
  gap: 12px;
  position: relative;
  z-index: 2;
}

.char-avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background-color: v-bind(colorBgAvatar);
  object-fit: cover;
  position: relative;
  z-index: 2;
}

.char-name {
  font-weight: bold;
  text-shadow: 1px 1px 3px v-bind(colorTextShadow);
}

.pull-info {
  display: flex;
  align-items: center;
  gap: 10px;
  position: relative;
  z-index: 2;
}

.pull-count {
  font-size: 1.2rem;
  font-weight: bold;
  color: v-bind(colorBrandPrimary);
  min-width: 30px;
  text-align: right;
  text-shadow: 1px 1px 3px v-bind(colorTextShadow);
}

/* 完整抽卡历史 */
.full-history-section {
  margin-top: 32px;
  padding-top: 24px;
  border-top: 1px solid v-bind(colorBgLighter);
}

.section-title {
  font-size: 1.1rem;
  color: v-bind(colorTextSecondary);
  margin-bottom: 16px;
}

.full-history-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  min-height: 631px;
}

.no-history-text {
  color: v-bind(colorTextTertiary);
  text-align: center;
  padding: 20px 0;
}

.full-history-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background-color: v-bind(colorBgLight);
  padding: 8px 12px;
  border-radius: 8px;
  border-left: 4px solid transparent;
}

/* 不同稀有度的左边框颜色 */
.full-history-item.SP {
  border-left-color: v-bind(colorRarityUr);
}

.full-history-item.SSR {
  border-left-color: v-bind(colorRaritySsr);
}

.full-history-item.SR {
  border-left-color: v-bind(colorRaritySr);
}

.full-history-item.R {
  border-left-color: v-bind(colorRarityR);
}

/* 不同稀有度的文字颜色 */
.rarity-SP {
  color: v-bind(colorRarityUr);
  font-weight: bold;
}

.rarity-SSR {
  color: v-bind(colorRaritySsr);
  font-weight: bold;
}

.rarity-SR {
  color: v-bind(colorRaritySr);
  font-weight: bold;
}

.rarity-R {
  color: v-bind(colorRarityR);
  font-weight: bold;
}

/* 分页控制 */
.pagination-controls {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 16px;
  margin-top: 24px;
  color: v-bind(colorTextSecondary);
  font-size: 0.9rem;
}

/* [NEW] 新增分页输入框样式 */
.pagination-controls span {
  display: inline-flex;
  align-items: center;
  gap: 8px;
}

.page-input {
  width: 50px;
  padding: 4px;
  text-align: center;
  background-color: v-bind(colorBgLight);
  color: v-bind(colorTextPrimary);
  border: 1px solid v-bind(colorBgLighter);
  border-radius: 4px;
  font-size: inherit;
}

.page-input:focus {
  outline: none;
  border-color: v-bind(colorBrandPrimary);
}

/* 隐藏数字输入框的上下箭头 */
.page-input::-webkit-outer-spin-button,
.page-input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

.page-input[type=number] {
  appearance: textfield;
  -moz-appearance: textfield;
}

.pagination-controls button {
  background-color: v-bind(colorBgLighter);
  color: v-bind(colorTextLight);
  border: none;
  padding: 8px 16px;
  border-radius: 6px;
  cursor: pointer;
  font-weight: bold;
  transition: background-color 0.2s;
}

.pagination-controls button:hover:not(:disabled) {
  background-color: v-bind(colorBgHover);
}

.pagination-controls button:disabled {
  background-color: v-bind(colorBgLight);
  color: v-bind(colorTextDisabled);
  cursor: not-allowed;
}
</style>
