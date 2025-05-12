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
    const reply = data.candidates?.[0]?.content?.parts?.[0]?.text ?? "描述不可用，请检查你的API密钥";
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
    const originalLife = life.value;
    if (originalLife.includes(" ❤️ ")) {
      life.value = originalLife.replace(" ❤️ ", " 💔 ");
    }

    if (!life.value.includes(" ❤️ ")) {
      gameOver.value = true;
      message.value = "😢 游戏失败！单词是：" + targetWord.value;
      return;
    }
  }

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

onMounted(() => {
  const input = document.getElementById("letter");
  window.addEventListener("keydown", (event) => {
    const key = event.key;
    if (/^[a-zA-Z]$/.test(key)) {
      input.value = key.toLowerCase();
    }
  });
});
</script>

<template>
  <main>
    <div id="display">{{ displayWord }}</div>
    <div id="life">{{ life }}</div>
    <div class="description">{{ description }}</div>
    <div v-if="gameOver" id="message">{{ message }}</div>
    <form onsubmit="return false">
      <div class="input_area">
        <span style="font-weight: 700">Guess a letter:</span>
        <input type="text" id="letter" maxlength="1" />
        <button id="guess" @click="guessLetter">Guess</button>
        <button id="Restart" @click="restartGame">Restart</button>
        <button id="description" @click="sendToAI">Get Description</button>
      </div>
    </form>
    <div class="apikey-container">
      <input
        type="password"
        id="apikey"
        placeholder="If you want git a hit, leave your Gemini API key here"
        v-model="apiKey"
      />
    </div>
    <div class="api-note">
      你的 API 密钥仅保存在浏览器内存中，不会以任何形式发送给第三方。<br />
      Your API key is only saved in your browser's memory and is not sent to
      third parties in any form.
    </div>
  </main>
</template>

<style scoped>
* {
  font-family: "Comic Sans MS", "Marker Felt", cursive, sans-serif;
  box-sizing: border-box;
}

main {
  border: 2px solid black;
  border-radius: 10px;
  padding: 20px;
  max-width: 600px;
  margin: 50px auto;
  text-align: center;
}

#display {
  font-size: 2rem;
  margin-bottom: 20px;
  letter-spacing: 10px;
}

#life {
  font-size: 1.5rem;
  margin-bottom: 20px;
}

#message {
  font-size: 1rem;
  font-style: italic;
  margin-bottom: 20px;
}

.description {
  font-size: 1rem;
  margin: 20px auto;
  max-width: 80%;
  padding: 10px;
  border: 2px dashed black;
  border-radius: 5px;
  background-color: #fdfdfd;
  font-style: italic;
  white-space: pre-wrap;
}

.input_area {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 10px;
  margin-top: 10px;
}

#letter {
  width: 40px;
  height: 30px;
  text-align: center;
  font-size: 1rem;
  border: 2px solid black;
  border-radius: 5px;
}

#guess,
#Restart,
#description {
  height: 35px;
  border: 2px solid black;
  border-radius: 5px;
  background-color: white;
  cursor: pointer;
  font-weight: bold;
  padding: 0 10px;
}

#guess:hover,
#Restart:hover,
#description:hover {
  background-color: #f0f0f0;
}

.apikey-container {
  display: flex;
  justify-content: center;
  margin-top: 20px;
}

#apikey {
  width: 60%;
  height: 30px;
  border: 2px solid black;
  border-radius: 5px;
  padding: 5px;
  font-family: "Comic Sans MS", cursive;
  font-size: 1rem;
  text-align: center;
}

.api-note {
  margin-top: 10px;
  font-size: 0.75rem;
  color: #555;
  font-style: italic;
}
</style>
