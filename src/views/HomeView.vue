<script setup>
  import { ref } from "vue";
  import OpenAI from "openai";

  const openai = new OpenAI({ apiKey: import.meta.env.VITE_OPEN_AI_API_KEY, dangerouslyAllowBrowser: true });

  const message = ref("");

  const history = ref([])

  const bot = ref({})

  const audio = ref(false)

  const textGPT = async () => {
    history.value.push({
      role: 'user',
      content: message.value
    })

    const response = await openai.chat.completions.create({
      messages: history.value,
      model: 'gpt-3.5-turbo'
    })

    history.value.push(response.choices[0].message)
    bot.value = response.choices[0].message

    if(audio.value){
      audioGPT()
    }
  }

  const audioGPT = async () => {
    const response = await openai.audio.speech.create({
      model: "tts-1-hd",
      voice: "nova",
      input: bot.value.content,
      response_format: "mp3"
    })

    console.log(response)

    const audio = new Audio()

    response.blob().then(result => {
      audio.src = URL.createObjectURL(result)
      audio.play()
    })
  }

  const clear = () => {
    history.value = []
  }

</script>

<template>
  <div>
    <p v-for="i in history" :key="i.role">
      {{ i.content }}
    </p>

    <input type="text" v-model="message" placeholder="请输入你要发送的内容">
    <button @click="textGPT">发送</button>
    <input type="checkbox" v-model="audio">播放语音
    <button @click="audioGPT">重新播放</button>
    <button @click="clear">清空对话</button>
  </div>
</template>
