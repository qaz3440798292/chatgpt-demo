<script setup>
  import { ref } from "vue";
  import OpenAI from "openai";

  // 创建openai对象
  const openai = new OpenAI({
    apiKey: import.meta.env.VITE_OPEN_AI_API_KEY,
    dangerouslyAllowBrowser: true,
    baseURL: import.meta.env.VITE_OPEN_AI_BASE_URL
  });

  // 用户输入的内容
  const message = ref("");

  // 对话历史记录，上下文
  const history = ref([])

  // 机器人回复
  const bot = ref({})

  // 是否播放语音
  const audio = ref(false)

  // 文本对话请求
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

    // 判断语音功能是否开启
    if(audio.value){
      audioGPT()
    }
  }

  // 语音链接
  const url = ref("")

  // 语音对话请求
  const audioGPT = async () => {
    if (audio.value){
      // 创建audio对象进行音频播放
      const audio = new Audio()

      // 发送请求
      const response = await openai.audio.speech.create({
        model: "tts-1-hd",
        voice: "nova",
        input: bot.value.content,
        response_format: "mp3"
      })

      console.log(response)

      response.blob().then(result => {
        url.value = URL.createObjectURL(result)
        audio.src = url.value
        audio.play()
      })
    }else{
      audio.value = confirm('你没有开启"播放语音"功能，你是否需要开启此功能？');
    }
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
    <a target="_blank" :href="url" download="speech.mp3">下载音频</a>
  </div>
</template>
