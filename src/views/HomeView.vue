<script setup>
  import { ref } from "vue";
  import OpenAI from "openai";

  // 创建openai对象
  const openai = new OpenAI({
    apiKey: import.meta.env.VITE_OPEN_AI_API_KEY, // apiKey请求后端所需的token
    dangerouslyAllowBrowser: true,
    baseURL: import.meta.env.VITE_OPEN_AI_BASE_URL // 修改默认URL
  });

  // 用户输入的内容
  const message = ref("");

  // 对话历史记录，上下文
  const history = ref([])

  // 机器人回复
  const bot = ref({})

  // 是否播放语音
  const audio = ref(false)

  // 允许发送
  const send = ref(true)

  // 文本对话请求
  const textGPT = async () => {
    history.value.push({
      role: 'user',
      content: message.value
    })

    console.log(message.value);

    const response = await openai.chat.completions.create({
      messages: history.value,
      model: 'gpt-3.5-turbo'
    })

    history.value.push(response.choices[0].message)
    bot.value = response.choices[0].message

    // 判断语音功能是否开启
    if(audio.value){
      await audioGPT()
    }

    send.value = !send.value
  }

  // 语音链接
  const url = ref("")

  // 语音对话请求
  const audioGPT = async () => {
    if (audio.value){
      // 创建audio对象进行音频播放
      const audio = new Audio()

      send.value = false

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

  let mediaRecoder // 麦克风
  let audioChunks // 录制的结果
  let media = false // 录制状态

  // 音频录制
  const mediaDevices = async () => {
    audioChunks = []

    if (!media){
      media = !media
      audio.value = true

      await navigator.mediaDevices.getUserMedia({ audio: true })
          .then(stream => {
            mediaRecoder = new MediaRecorder(stream)
            mediaRecoder.ondataavailable = e => {
              audioChunks.push(e.data)
            }

            //录制停止事件
            mediaRecoder.onstop = async () => {
              const audioBlob = new Blob(audioChunks, { type: 'audio/mp3' })

              const formData = new FormData()

              formData.append('file', audioBlob, 'audio.mp3')

              send.value = false

              const response = await openai.audio.transcriptions.create({
                file: formData.get('file'),
                model: 'whisper-1'
              })

              message.value = response.text

              console.log(response.text);
              await textGPT()
            }

            //录制开始
            mediaRecoder.start()
          })
          // 麦克风无授权
          .catch(err => {
            console.error('你未进行语音授权，无法使用语音功能', err)
            alert('你未进行语音授权，无法使用语音功能')
          })

    }else {
      media = !media

      // 停止录制
      mediaRecoder.stop()

      // 停止所有录制
      mediaRecoder.stream.getTracks().forEach( track => {
        track.stop()
      })
    }
  }

</script>

<template>
  <div>
    <p v-for="i in history" :key="i.role">
      {{ i.content }}
    </p>

    <input type="text" v-model="message" placeholder="请输入你要发送的内容">
    <button @click="textGPT" :disabled="!send">发送</button>
    <input type="checkbox" v-model="audio">播放语音
    <button @click="audioGPT">重新播放</button>
    <button @click="clear">清空对话</button>
    <button @click="mediaDevices" :disabled="!send">通话</button>
    <a target="_blank" :href="url" download="speech.mp3">下载音频</a>
  </div>
</template>
