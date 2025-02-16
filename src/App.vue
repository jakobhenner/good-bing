<script setup>
  import { ref, onMounted, onUnmounted } from "vue";
  import { Howl } from "howler";

  // State variables
  const transcription = ref([]);
  const sentences = ref([]);
  const currentTime = ref(0);
  const currentSentenceIndex = ref(null);
  const audio = ref(null);
  const showPlayButton = ref(true);
  let animationFrame = null;

  // Load transcription JSON
  const fetchTranscription = async () => {
    const response = await fetch("/transcription.json");
    const words = await response.json();

    // Group words into sentences based on punctuation
    let tempSentence = [];
    let sentenceStart = 0;
    sentences.value = [];

    words.forEach((word, index) => {
      tempSentence.push(word);
      if (
        word.word.includes(".") ||
        word.word.includes("?") ||
        word.word.includes("!")
      ) {
        sentences.value.push({
          text: tempSentence.map((w) => w.word).join(" "),
          start_time: sentenceStart,
          end_time: word.end_time,
          words: tempSentence,
        });
        tempSentence = [];
        if (index + 1 < words.length)
          sentenceStart = words[index + 1].start_time;
      }
    });

    if (tempSentence.length) {
      sentences.value.push({
        text: tempSentence.map((w) => w.word).join(" "),
        start_time: sentenceStart,
        end_time: tempSentence[tempSentence.length - 1].end_time,
        words: tempSentence,
      });
    }
  };

  // Play audio & track time efficiently
  const playAudio = () => {
    if (!audio.value) {
      audio.value = new Howl({
        src: ["/good-bing.mp3"],
        format: ["mp3"],
        html5: true,
        onplay: updateTime,
        onseek: updateTime,
        onend: () => {
          showPlayButton.value = true;
          currentSentenceIndex.value = null;
        },
      });
    }
    showPlayButton.value = false;
    audio.value.play();
  };

  // Use requestAnimationFrame for better performance
  const updateTime = () => {
    currentTime.value = audio.value.seek();

    // Find the currently active sentence, but allow gaps to be "inactive"
    const newIndex = sentences.value.findIndex(
      (sentence) =>
        sentence.start_time <= currentTime.value &&
        sentence.end_time >= currentTime.value
    );

    // If nothing is active, set null instead of keeping the last sentence
    currentSentenceIndex.value = newIndex !== -1 ? newIndex : null;

    animationFrame = requestAnimationFrame(updateTime);
  };

  const getSentenceClass = (index) => {
    if (index === currentSentenceIndex.value) return "active";
  };

  // Stop animation on unmount
  onUnmounted(() => {
    if (animationFrame) cancelAnimationFrame(animationFrame);
  });

  // Load data on mount
  onMounted(fetchTranscription);
</script>

<template>
  <div class="container">
    <transition name="fade" mode="out-in" duration="400" appear>
      <button @click="playAudio" v-if="showPlayButton">
        Have you been a good Bing?
      </button>
      <div v-else>
        <div class="lyrics">
          <div
            v-for="(sentence, index) in sentences"
            :key="index"
            class="sentence"
            :class="getSentenceClass(index)"
          >
            <span
              v-for="(word, i) in sentence.words"
              :key="i"
              class="word"
              :class="{
                highlight:
                  currentTime >= word.start_time &&
                  index === currentSentenceIndex,
              }"
            >
              {{ word.word }}
            </span>
          </div>
        </div>
      </div>
    </transition>
  </div>
</template>

<style>
  button {
    background: #fff;
    color: transparent;
    appearance: none;
    border: none;
    cursor: pointer;
    font: inherit;
    letter-spacing: inherit;
    position: absolute;
    top: 0;
    left: 0;
    inset: 0;
    text-align: left;
    display: flex;
    align-items: flex-start;
    justify-content: flex-start;
    padding: 1rem;
    background-image: linear-gradient(
      90deg,
      rgba(0, 0, 0, 0.5) 0%,
      #000 25%,
      #000 50%,
      rgba(0, 0, 0, 0.5) 75%,
      rgba(0, 0, 0, 0.5) 100%
    );
    background-size: 200% 100%;
    background-clip: text;
    -webkit-background-clip: text;
    animation: shine 3s linear infinite;
  }

  @keyframes shine {
    0% {
      background-position: 100% 0;
    }
    100% {
      background-position: -100% 0;
    }
  }
  .container {
    padding: 1rem;
  }

  .lyrics {
    overflow: hidden;
    color: rgba(0, 0, 0, 0.3);
    transition: color 0.3s ease-in-out;
    display: grid;
    gap: 0.75rem;
    text-wrap: balance;
    max-width: 22rem;
  }

  .sentence {
    transition: opacity 0.4s ease-in-out, filter 0.4s ease-in-out;
    opacity: 0;
  }
  .sentence.active {
    color: rgba(0, 0, 0, 0.5);
    opacity: 1;
  }

  .word {
    transition: color 0.2s ease-in-out;
  }

  .highlight {
    color: #000;
  }

  /* Fade transition */
  .fade-enter-active,
  .fade-leave-active {
    transition: opacity 0.4s ease-in-out;
  }

  .fade-enter-from,
  .fade-leave-to {
    opacity: 0;
  }
</style>
