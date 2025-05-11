<script setup>
import { ref, onMounted, computed } from "vue";

const letter = ref("");
const message = ref("");
const life = ref("");
const targetWord = ref("");
const revealedIndices = ref(new Set());
const gameOver = ref(false);
const description = ref(" ");
const apiKey = ref("");

const sendToAI = async () => {
  const key = apiKey.value;
  if (!key) {
    alert(
      "请先输入API密钥以使用Gemini功能, 获取api的链接在这里 https://aistudio.google.com/app/apikey?hl=zh-cn"
    );
    return;
  }
  const url = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${key}`;

  const body = {
    contents: [
      {
        parts: [
          {
            text: `你是一个词典老师，用中文解释 "${targetWord.value}" 这个单词的意思。不能给出这个单词的拼音或英文翻译,不能把这个词以任何形式出现在你的回答中。用最简短的语言来描述`,
          },
        ],
      },
    ],
  };
  try {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });
    const data = await response.json();
    const reply = data.candidates?.[0]?.content?.parts?.[0]?.text ?? "无回应";
    console.log("Gemini response:", reply);
    description.value = reply;
  } catch (error) {
    console.error("Error:", error);
  }
};

const displayWord = computed(() => {
  return targetWord.value
    .split("")
    .map((char, index) => (revealedIndices.value.has(index) ? char : "_"))
    .join(" ");
});

const fetchNewWord = async () => {
  try {
    const res = await fetch("https://random-word-api.vercel.app/api");
    const data = await res.json();
    targetWord.value = data[0];
    console.log("Fetched word:", targetWord.value);
    life.value = " ❤️ ".repeat(targetWord.value.length * 2);
    const randomIndex = Math.floor(Math.random() * targetWord.value.length);
    revealedIndices.value.add(randomIndex);
  } catch (err) {
    console.error("Failed to fetch word:", err);
  }
};

const guessLetter = () => {
  if (gameOver.value) return;

  const input = document.getElementById("letter");
  const letter = input.value.trim().toLowerCase();
  if (!letter) {
    input.value = "";
    return;
  }

  // 找出匹配该字母但尚未揭示的位置
  const matchedIndices = targetWord.value
    .split("") // sample output: ["h", "e", "l", "l", "o"]
    .map((char, index) => ({ char, index })) // sample output: [{char: "h", index: 0}, {char: "e", index: 1}, ...]
    .filter(
      ({ char, index }) =>
        char.toLowerCase() === letter && !revealedIndices.value.has(index)
    ) // sample output: [{char: "l", index: 2}, {char: "l", index: 3}]
    .map(({ index }) => index); // sample output: [2, 3]

  // 如果有未揭示的位置，从中随机选一个揭示
  if (matchedIndices.length > 0) {
    const randomReveal =
      matchedIndices[Math.floor(Math.random() * matchedIndices.length)];
    revealedIndices.value.add(randomReveal);
  } else {
    console.log("No matching letter found. Current life:", life.value);
    const originalLife = life.value;
    if (originalLife.includes(" ❤️ ")) {
      life.value = originalLife.replace(" ❤️ ", " 💔 ");
      console.log("Life updated:", life.value);
    }

    if (!life.value.includes(" ❤️ ")) {
      gameOver.value = true;
      message.value = "😢 游戏失败！单词是：" + targetWord.value;
      return;
    }
  }

  message.value = `正确的字母: ${letter}`;

  // 检查是否所有字母都已揭示
  const allRevealed = targetWord.value
    .split("")
    .every((_, index) => revealedIndices.value.has(index));

  if (allRevealed) {
    gameOver.value = true;
    message.value = "🎉 恭喜你！你猜出了单词！";
  }

  input.value = "";
  input.focus();
};

const restartGame = () => {
  letter.value = "";
  message.value = "";
  life.value = "";
  revealedIndices.value.clear();
  gameOver.value = false;
  targetWord.value = "";
  description.value = " ";
  fetchNewWord();
};

onMounted(async () => {
  fetchNewWord();
});
</script>

<template>
  <main>
    <div id="display">{{ displayWord }}</div>
    <div id="life">{{ life }}</div>
    <div v-if="gameOver" id="message">{{ message }}</div>
    <div class="description">{{ description }}</div>
    <form onsubmit="return false">
      <div class="input_area">
        <span style="font-weight: 700">Guess a letter:</span>
        <input type="text" id="letter" maxlength="1" />
        <button id="guess" @click="guessLetter">Guess</button>
        <button id="Restart" @click="restartGame">Restart</button>
        <button id="description" @click="sendToAI">Get Description</button>
      </div>
    </form>
    <div style="display: flex; justify-content: center; margin-top: 2rem">
      <input
        type="password"
        id="apikey"
        placeholder="If you want git a hit, leave your Gemini API key here"
        style="width: 40%"
        v-model="apiKey"
      />
    </div>
  </main>
</template>

<style scoped>
* {
  font-family: Arial, Helvetica, sans-serif;
}

.main {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100dvh;
}

#life {
  margin-left: auto;
  margin-right: auto;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 1rem;
  font-weight: 700;
  margin-top: 2rem;
  color: #f44336;
  text-shadow: 2px 2px 4px #000000;
}
#message {
  margin-left: auto;
  margin-right: auto;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 1rem;
  font-weight: 700;
  margin-top: 2rem;
  color: #4caf50;
}

#display {
  margin-left: auto;
  margin-right: auto;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 2rem;
  font-weight: 700;
  margin-top: 2rem;
  color: #4caf50;
  text-shadow: 2px 2px 4px #000000;
  background-color: #f0f0f0;
  padding: 20px;
  border-radius: 10px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  min-width: 300px;
  max-width: 1000px;
  height: 20px;
  border: 2px solid #4caf50;
  transition: all 0.3s ease;
}

.input_area {
  flex: 0 0 1;
  display: flex;
  justify-content: center;
  align-items: center;
  margin-top: 2rem;
  gap: 2%;
}

#letter,
#apikey {
  width: 100px;
  height: 30px;
  border-radius: 5px;
  padding-left: 5px;
}

#guess {
  width: 100px;
  height: 30px;
  border-radius: 5px;
  border: none;
  background-color: #4caf50;
  color: white;
}
#guess:hover {
  background-color: #45a049;
}

#Restart {
  width: 100px;
  height: 30px;
  border-radius: 5px;
  border: none;
  background-color: #f44336;
  color: white;
}
#Restart:hover {
  background-color: #e53935;
}

#description {
  width: 100px;
  height: 30px;
  border-radius: 5px;
  border: none;
  background-color: #2196f3;
  color: white;
}
#description:hover {
  background-color: #1976d2;
}

.description {
  font-size: 1.25rem;
  color: #333;
  text-align: center;
  max-width: 60%;
  background-color: #f0f0f0;
  padding: 20px;
  border-radius: 10px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  margin-left: auto;
  margin-right: auto;
}
</style>
