<template>
  <!-- 最外層背景 (燕麥暖白) -->
  <div
    class="min-h-screen bg-[#FDFBF7] text-gray-700 flex flex-col justify-between items-center p-4"
  >
    <!-- 1. Header 區域 -->
    <header class="w-full max-w-sm flex flex-col items-center pt-2 pb-1">
      <div class="text-3xl mb-1 select-none">🐱</div>
      <span class="text-xs text-gray-400 font-medium">陪伴貓咪小助手</span>
    </header>

    <!-- 2. 主內容卡片區域 -->

    <!-- TAB 1: 聊天室 -->
    <main
      v-if="currentTab === 'chat'"
      ref="chatContainer"
      class="w-full max-w-sm flex-1 bg-white/70 backdrop-blur-sm rounded-3xl p-5 my-3 shadow-sm border border-stone-100 flex flex-col gap-4 overflow-y-auto max-h-[65vh] scroll-smooth"
    >
      <div
        v-for="(msg, index) in messages"
        :key="index"
        :class="[
          'max-w-[85%] text-sm leading-relaxed whitespace-pre-line',
          msg.role === 'user'
            ? 'self-end bg-[#EFECE6] text-stone-800 px-4 py-2.5 rounded-2xl'
            : 'self-start text-stone-700',
        ]"
      >
        <span
          v-if="msg.content === '貓咪思考中喵...'"
          class="animate-pulse flex items-center gap-1 text-stone-400"
        >
          🐾 貓咪正在思考中...
        </span>
        <span v-else>
          {{ msg.content }}
        </span>
      </div>
    </main>

    <!-- TAB 2: 我 & AI -->
    <main
      v-else-if="currentTab === 'profile'"
      class="w-full max-w-sm flex-1 bg-white/70 backdrop-blur-sm rounded-3xl p-5 my-3 shadow-sm border border-stone-100 flex flex-col gap-4 overflow-y-auto max-h-[65vh]"
    >
      <h2
        class="text-base font-bold text-stone-700 border-b border-stone-100 pb-2 flex items-center gap-2"
      >
        <span>👤</span> 我 & AI 個人設定
      </h2>

      <div class="space-y-4 text-sm">
        <div>
          <label class="block text-xs font-medium text-stone-500 mb-1"
            >目標體重 (kg)</label
          >
          <input
            v-model="userProfile.targetWeight"
            type="number"
            step="0.1"
            class="w-full bg-white border border-stone-200 rounded-xl px-3 py-2 text-stone-700 outline-none focus:border-amber-400 transition"
            placeholder="例如: 65.0"
          />
        </div>

        <div>
          <label class="block text-xs font-medium text-stone-500 mb-1"
            >陪伴貓咪性格</label
          >
          <select
            v-model="userProfile.catPersonality"
            class="w-full bg-white border border-stone-200 rounded-xl px-3 py-2 text-stone-700 outline-none focus:border-amber-400 transition"
          >
            <option value="gentle">溫柔治癒系 🐾</option>
            <option value="strict">嚴格監督系 😾</option>
            <option value="cheerful">活潑熱情系 😸</option>
          </select>
        </div>

        <button
          @click="saveProfile"
          class="w-full py-2.5 bg-[#A89885] text-white rounded-xl font-medium hover:opacity-90 active:scale-98 transition shadow-sm"
        >
          儲存設定
        </button>
      </div>
    </main>

    <!-- TAB 3: 記錄 (體重數據與趨勢圖表) -->
    <main
      v-else-if="currentTab === 'records'"
      class="w-full max-w-sm flex-1 bg-white/70 backdrop-blur-sm rounded-3xl p-5 my-3 shadow-sm border border-stone-100 flex flex-col gap-4 overflow-y-auto max-h-[65vh]"
    >
      <h2
        class="text-base font-bold text-stone-700 border-b border-stone-100 pb-2 flex items-center justify-between"
      >
        <span class="flex items-center gap-2"><span>📋</span> 體重記錄</span>
        <button
          @click="clearRecords"
          class="text-xs font-normal text-rose-400 hover:text-rose-600"
        >
          清空數據
        </button>
      </h2>

      <!-- 最新狀態卡片 -->
      <div
        class="bg-amber-50/60 border border-amber-100 rounded-2xl p-4 flex justify-between items-center"
      >
        <div>
          <div class="text-xs text-stone-400 mb-1">最新記錄體重</div>
          <div class="text-2xl font-bold text-stone-800">
            {{ latestWeight ? `${latestWeight} kg` : "尚未記錄" }}
          </div>
        </div>
        <div class="text-right">
          <div class="text-xs text-stone-400 mb-1">距離目標</div>
          <div class="text-sm font-semibold text-amber-700">
            {{ weightDiff }}
          </div>
        </div>
      </div>

      <!-- 折線圖區域 -->
      <div class="bg-white p-3 rounded-2xl border border-stone-100">
        <div class="text-xs font-medium text-stone-400 mb-2">
          體重變化趨勢圖
        </div>
        <div class="h-44 w-full flex items-center justify-center">
          <Line
            v-if="weightRecords.length > 0"
            :data="chartData"
            :options="chartOptions"
          />
          <div v-else class="text-xs text-stone-400">尚無足夠數據繪製圖表</div>
        </div>
      </div>

      <!-- 體重歷史列表 -->
      <div class="flex flex-col gap-2">
        <div class="text-xs font-medium text-stone-400 mb-1">
          歷史紀錄 (對話自動提取)
        </div>
        <div
          v-for="(rec, i) in weightRecords"
          :key="i"
          class="bg-white p-3 rounded-xl border border-stone-100 flex justify-between items-center text-sm"
        >
          <span class="text-stone-400 text-xs">{{ rec.date }}</span>
          <span class="font-bold text-stone-700">{{ rec.weight }} kg</span>
        </div>
        <div
          v-if="weightRecords.length === 0"
          class="text-center text-xs text-stone-400 py-6"
        >
          在聊天室告訴貓咪體重（如：我今天70.5），會自動記錄在這裡喔！
        </div>
      </div>
    </main>

    <!-- 3. 輸入框區域 (僅在聊天室顯示) -->
    <footer class="w-full max-w-sm flex flex-col gap-3">
      <form
        v-if="currentTab === 'chat'"
        @submit.prevent="sendMessage"
        class="bg-white rounded-full p-1.5 pl-4 shadow-sm border border-stone-200/60 flex items-center justify-between"
      >
        <input
          v-model="inputMessage"
          :disabled="isLoading"
          type="text"
          placeholder="輸入訊息，如：我今天體重70.5"
          class="w-full text-sm bg-transparent border-none outline-none text-stone-700 placeholder-stone-400 disabled:opacity-50"
        />
        <button
          type="submit"
          :disabled="isLoading"
          class="w-10 h-10 rounded-full bg-[#A89885] text-white flex items-center justify-center text-sm shadow-sm hover:opacity-90 active:scale-95 transition-all flex-shrink-0 cursor-pointer disabled:opacity-50"
        >
          🐾
        </button>
      </form>

      <!-- 4. Bottom Navigation 頁籤切換 -->
      <nav
        class="flex justify-around items-center py-2 text-xs text-stone-400 bg-white/40 rounded-full border border-stone-100"
      >
        <button
          @click="switchTab('chat')"
          :class="[
            'flex flex-col items-center gap-1 transition-all',
            currentTab === 'chat'
              ? 'text-stone-800 font-bold scale-105'
              : 'hover:text-stone-600',
          ]"
        >
          <span>💬</span>
          <span>聊天室</span>
        </button>
        <button
          @click="switchTab('profile')"
          :class="[
            'flex flex-col items-center gap-1 transition-all',
            currentTab === 'profile'
              ? 'text-stone-800 font-bold scale-105'
              : 'hover:text-stone-600',
          ]"
        >
          <span>👤</span>
          <span>我 & AI</span>
        </button>
        <button
          @click="switchTab('records')"
          :class="[
            'flex flex-col items-center gap-1 transition-all',
            currentTab === 'records'
              ? 'text-stone-800 font-bold scale-105'
              : 'hover:text-stone-600',
          ]"
        >
          <span>📋</span>
          <span>記錄</span>
        </button>
      </nav>
    </footer>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch, nextTick } from "vue";

// 引入 Chart.js 組件與設定
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
} from "chart.js";
import { Line } from "vue-chartjs";

ChartJS.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
);

// --- 狀態定義 ---
const currentTab = ref("chat");
const chatContainer = ref(null);

const messages = ref([
  { role: "model", content: "早安，我的小貓☀️\n今天有量空腹體重嗎？" },
]);

const inputMessage = ref("");
const isLoading = ref(false);

const userProfile = ref({
  targetWeight: 65.0,
  catPersonality: "gentle",
});

const weightRecords = ref([
  { date: "2026/09/04 03:58", weight: 72.3 },
  { date: "2026/09/04 03:59", weight: 71.3 },
]);

// --- LocalStorage 持久化 ---
onMounted(async () => {
  const savedMessages = localStorage.getItem("cat_chat_messages");
  if (savedMessages) messages.value = JSON.parse(savedMessages);

  const savedProfile = localStorage.getItem("cat_user_profile");
  if (savedProfile) userProfile.value = JSON.parse(savedProfile);

  const savedRecords = localStorage.getItem("cat_weight_records");
  if (savedRecords) weightRecords.value = JSON.parse(savedRecords);

  // 確保 DOM 載入與資料更新完畢後自動滾動到底部
  await nextTick();
  scrollToBottom();
});

watch(
  messages,
  (newVal) => {
    localStorage.setItem("cat_chat_messages", JSON.stringify(newVal));
  },
  { deep: true },
);

watch(
  userProfile,
  (newVal) => {
    localStorage.setItem("cat_user_profile", JSON.stringify(newVal));
  },
  { deep: true },
);

watch(
  weightRecords,
  (newVal) => {
    localStorage.setItem("cat_weight_records", JSON.stringify(newVal));
  },
  { deep: true },
);

// --- 計算屬性 ---
const latestWeight = computed(() => {
  if (weightRecords.value.length === 0) return null;
  return weightRecords.value[weightRecords.value.length - 1].weight;
});

const weightDiff = computed(() => {
  if (!latestWeight.value || !userProfile.value.targetWeight)
    return "未設定目標";
  const diff = (latestWeight.value - userProfile.value.targetWeight).toFixed(1);
  if (diff > 0) return `還差 ${diff} kg`;
  if (diff == 0) return "已達標！🎉";
  return `低於目標 ${Math.abs(diff)} kg`;
});

// --- 圖表數據與設定 ---
const chartData = computed(() => {
  return {
    labels: weightRecords.value.map((r) => r.date.split(" ")[0]), // 只顯示日期 YYYY/MM/DD
    datasets: [
      {
        label: "體重 (kg)",
        backgroundColor: "#A89885",
        borderColor: "#A89885",
        pointBackgroundColor: "#D97706",
        pointRadius: 4,
        tension: 0.3, // 平滑曲線
        data: weightRecords.value.map((r) => r.weight),
      },
    ],
  };
});

const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: { display: false },
    tooltip: {
      callbacks: {
        label: (context) => `${context.raw} kg`,
      },
    },
  },
  scales: {
    x: {
      grid: { display: false },
      ticks: { font: { size: 10 }, color: "#A8A29E" },
    },
    y: {
      grid: { color: "#F5F5F4" },
      ticks: { font: { size: 10 }, color: "#A8A29E" },
    },
  },
};

// --- 方法 ---
const switchTab = (tab) => {
  currentTab.value = tab;
  if (tab === "chat") {
    scrollToBottom();
  }
};

const scrollToBottom = async () => {
  await nextTick();
  if (chatContainer.value) {
    chatContainer.value.scrollTop = chatContainer.value.scrollHeight;
  }
};

const saveProfile = () => {
  alert("設定已成功儲存喵！🐾");
};

const clearRecords = () => {
  if (confirm("確定要清空所有體重紀錄嗎？")) {
    weightRecords.value = [];
  }
};

function parseWeight(inputText) {
  // 1. 排除明確不是體重的單位與文字（例如：分鐘、公里、步、元、歲、cm...）
  if (/(分鐘|分|公里|km|m|步|元|塊|歲|年|月|日|號|cm|公分)/i.test(inputText)) {
    // 若文字中帶有其他單位，但完全沒有寫到「kg/公斤/體重」，則直接忽略
    if (!/(kg|公斤|體重|重)/i.test(inputText)) {
      return null;
    }
  }

  // 2. 精確捕捉體重：尋找「體重/重/公斤/kg」旁邊的數字
  const keywordMatch = inputText.match(
    /(?:體重|重|是)?\s*(\d{2,3}(?:\.\d{1,2})?)\s*(?:kg|公斤|公斤重)?/i,
  );

  if (keywordMatch && keywordMatch[1]) {
    const val = parseFloat(keywordMatch[1]);
    // 限制合理的體重範圍，避免非體重數字誤入
    if (val >= 30 && val <= 200) {
      return val; // 直接回傳解析出來的數字
    }
  }

  return null;
}

// 自動解析體重與時間格式化
const extractAndSaveWeight = (text) => {
  const weightNum = parseWeight(text); // ✅ 直接接收數字或 null

  if (weightNum !== null) {
    const now = new Date();
    const year = now.getFullYear();
    const month = String(now.getMonth() + 1).padStart(2, "0");
    const date = String(now.getDate()).padStart(2, "0");
    const hours = String(now.getHours()).padStart(2, "0");
    const minutes = String(now.getMinutes()).padStart(2, "0");

    const timeStr = `${year}/${month}/${date} ${hours}:${minutes}`;

    weightRecords.value.push({
      date: timeStr,
      weight: weightNum,
    });
  }
};

// 發送訊息
const sendMessage = async () => {
  if (!inputMessage.value.trim() || isLoading.value) return;

  const userText = inputMessage.value.trim();

  // 1. 解析並紀錄體重
  extractAndSaveWeight(userText);

  messages.value.push({ role: "user", content: userText });
  inputMessage.value = "";
  isLoading.value = true;

  messages.value.push({ role: "model", content: "貓咪思考中喵..." });
  scrollToBottom();

  try {
    const apiKey = import.meta.env.VITE_GEMINI_API_KEY;
    const todayStr = new Date().toLocaleDateString("zh-TW");

    const personalityMap = {
      gentle: "溫柔、治癒且體貼",
      strict: "稍微傲嬌但非常關心健康的嚴格",
      cheerful: "超級熱情活潑、滿滿正能量",
    };
    const currentStyle =
      personalityMap[userProfile.value.catPersonality] || "溫柔陪伴";

    const systemInstruction = `你是生活陪伴貓咪助手。
今天的日期是：${todayStr}。
性格風格：${currentStyle}。

注意事項：
1. 請用簡短、治癒的口吻（帶貓咪表情符號）回覆使用者。
2. 使用者可能會在同一天紀錄多次體重數據，進行數據比較時請稱呼為「上一筆紀錄」或「剛才」，除非確認日期不同，否則不要輕易使用「昨天」或「上次」。`;

    const contents = messages.value.slice(0, -1).map((m) => ({
      role: m.role === "user" ? "user" : "model",
      parts: [{ text: m.content }],
    }));

    // 2. 修正 API 模型名稱 (gemini-1.5-flash)
    const response = await fetch(
      `https://generativelanguage.googleapis.com/v1beta/models/gemini-3.6-flash:generateContent?key=${apiKey}`,
      {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          systemInstruction: { parts: [{ text: systemInstruction }] },
          contents: contents,
        }),
      },
    );

    const data = await response.json();

    if (data.candidates && data.candidates[0]?.content?.parts[0]?.text) {
      messages.value[messages.value.length - 1].content =
        data.candidates[0].content.parts[0].text;
    } else {
      messages.value[messages.value.length - 1].content =
        "喵嗚...連線有點慢，再試一次看看？";
    }
  } catch (error) {
    messages.value[
      messages.value.length - 1
    ].content = `喵～小狀況：${error.message}`;
  } finally {
    isLoading.value = false;
    scrollToBottom();
  }
};
</script>
