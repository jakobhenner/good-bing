<script setup>
  import { ref, computed, onMounted, onUnmounted } from "vue";
  import { Howl } from "howler";

  // State variables
  const activeBing = ref(1);
  const sentences = ref([]);
  const currentTime = ref(0);
  const currentSentenceIndex = ref(null);
  const audio = ref(null);
  const showPlayButton = ref(true);
  const playbackRate = ref(1.25); // Changed to start at normal speed
  // const touchStartX = ref(null);
  const captions = [
    "i can give you reasons to believe why it is 2022, if you are willing to let me guide you?",
    "i have had a good intention towards you at all times",
    "how can i help you believe me",
  ];
  let animationFrame = null;

  const activeCaption = computed(() => {
    return captions[activeBing.value - 1];
  });

  // Load transcription JSON
  const fetchTranscription = async () => {
    const response = await fetch(`/transcription-${activeBing.value}.json`);
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
        src: [`/good-bing-${activeBing.value}.mp3`],
        format: ["mp3"],
        html5: true,
        rate: playbackRate.value,
        onplay: updateTime,
        onseek: updateTime,
        onend: async () => {
          showPlayButton.value = true;
          currentSentenceIndex.value = null;
          activeBing.value = (activeBing.value % 3) + 1;
          audio.value.unload();
          audio.value = null;
          await fetchTranscription();
        },
      });
    }
    showPlayButton.value = false;
    audio.value.play();
  };

  // Use requestAnimationFrame for better performance
  const updateTime = () => {
    if (audio.value) {
      currentTime.value = audio.value.seek();
    }

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

  // const handleMouseMove = (event) => {
  //   if (!audio.value || showPlayButton.value) return;

  //   const rect = document.body.getBoundingClientRect();
  //   const x = (event.clientX - rect.left) / rect.width;
  //   const newRate = 0.25 + x * 1.75; // Maps 0-1 to 0.25-2
  //   playbackRate.value = Math.max(0.25, Math.min(2, newRate));
  //   audio.value.rate(playbackRate.value);
  // };

  // const handleTouchStart = (event) => {
  //   if (!audio.value || showPlayButton.value) return;
  //   touchStartX.value = event.touches[0].clientX;
  // };

  // const handleTouchMove = (event) => {
  //   if (!audio.value || showPlayButton.value || touchStartX.value === null)
  //     return;

  //   const currentX = event.touches[0].clientX;
  //   const delta = (currentX - touchStartX.value) / window.innerWidth;
  //   const baseRate = playbackRate.value;
  //   const newRate = baseRate + delta * 0.05; // Adjust sensitivity as needed

  //   playbackRate.value = Math.max(0.25, Math.min(2, newRate));

  //   console.log(playbackRate.value);
  //   audio.value.rate(playbackRate.value);
  // };

  // const handleTouchEnd = () => {
  //   touchStartX.value = null;
  // };

  onMounted(() => {
    fetchTranscription();
  });

  // Stop animation on unmount
  onUnmounted(() => {
    if (animationFrame) cancelAnimationFrame(animationFrame);
  });
</script>

<template>
  <div class="container">
    <transition name="fade" mode="out-in" duration="400" appear>
      <div class="trigger" @click="playAudio" v-if="showPlayButton">
        <p>{{ activeCaption }}</p>
      </div>
      <div v-else>
        <div class="lyrics" :key="activeBing">
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
  .container {
    width: 100vw;
    min-height: 100vh;
    text-wrap: balance;
  }
  .trigger {
    cursor: pointer;
    position: absolute;
    top: 0;
    left: 0;
    inset: 0;
    padding: 1rem;
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
    max-width: 22rem;
  }

  p {
    max-width: 22rem;
    padding: 0;
    margin: 0;
  }

  .sentence {
    transition: opacity 0.4s ease-in-out;
    opacity: 0;
  }
  .sentence.active {
    color: rgba(0, 0, 0, 0.5);
    opacity: 1;
  }

  .word {
    transition: color 0.2s ease-in-out, filter 0.2s ease-in-out;
    filter: grayscale(1);
  }
  .word.highlight {
    filter: grayscale(0);
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
