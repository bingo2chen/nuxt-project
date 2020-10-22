<template>
  <div>
    <h2>用户中心</h2>
    <i class="el-icon-loading"></i>
    <div id="drag" ref="drag">
      <input type="file" name="file" @change="handleFileChange">
    </div>
    <div>
      <p>文件上传进度条</p>
      <el-progress :text-inside="true" :stroke-width="26" :percentage="uploadProgress"></el-progress>
    </div>
    <div class="mt50">
      <p>跟踪hash进度条</p>
      <el-progress :text-inside="true" :stroke-width="26" :percentage="hashProgress"></el-progress>
    </div>
    <div>
      <!-- chunk.progress 网格进度条
      progress<0 报错 显示红色
              == 100 成功
              别的数字 方块高亮显示
      尽量让方块看起来像正方形
      比如 16个方块 4*4
           9个方块 3*3
           100个方块 10*10 -->
      <div class="cube-container" :style="{width:cubeWidth+'px'}">
        <div class="cube" v-for="chunk in chunks" :key="chunk.name">
          <div
            :class="{
              'uploading': chunk.progress>0&&chunk.progress<100,
              'success': chunk.progress===100,
              'error': chunk.progress<0
            }"
            :style="{height:chunk.progress+'%'}"
          >
            <i class="el-icon-loading" style="color:#f56c6c" v-if="chunk.progress<100&&chunk.progress>0"></i>
          </div>
        </div>
      </div>
    </div>
    <el-button @click="uploadFile">上传</el-button>
  </div>
</template>

<script>
  import sparkMD5 from 'spark-md5'
  const CHUNK_SIZE = 0.1 * 1024 * 1024
  export default {
    async mounted() {
      const ret = await this.$http.get('/user/info')
      console.log('ret', ret)

      this.bindEvent()
    },
    data() {
      return {
        file: null,
        // uploadProgress: 0,
        hashProgress: 0,
        chunks: []
      }
    },
    computed: {
      cubeWidth() {
        return Math.ceil(Math.sqrt(this.chunks.length)) * 16 
      },
      uploadProgress() { // 切片上传使用
        if (!this.file || this.chunks.length === 0) {
          return 0
        }
        const loaded = this.chunks.map(item => item.chunk.size * item.progress)
                                  .reduce((acc, cur) => acc + cur, 0)
                
        return parseInt(((loaded * 100) / this.file.size).toFixed(2))
      }
    },
    methods: {
      bindEvent() {
        const drag = this.$refs.drag
        drag.addEventListener('dragover', e => {
          drag.style.borderColor = 'yellowgreen'
          e.preventDefault()
        })
        drag.addEventListener('dragleave', e => {
          drag.style.borderColor = '#eee'
          e.preventDefault()
        })
        drag.addEventListener('drop', e => {
          const fileList = e.dataTransfer.files
          drag.style.borderColor = '#eee'
          this.file = fileList[0]
          console.log(this.file);
          
          e.preventDefault()
        })
      },
      async blobToString(blob) {
        return new Promise(resolve => {
          const reader = new FileReader()
          reader.onload = () => {
            console.log(reader.result);
            const ret = reader.result.split('')
                            .map(v => v.charCodeAt())
                            .map(v => v.toString(16).toUpperCase())
                            .join('')
            resolve(ret)
          }
          reader.readAsBinaryString(blob)
        })
      },
      async isGif(file) {
        // GIF89a 和 GIF87a
        // 前面6个16进制，'47 49 46 38 39 61'  '47 49 46 38 37 61'
        const ret = await this.blobToString(file.slice(0, 6))
        const isGif = (ret === '47 49 46 38 39 61') || (ret === '47 49 46 38 37 61')
        return isGif
      },
      async isPng(file) {
        const ret = await this.blobToString(file.slice(0, 8))
        const isGif = (ret === '89 50 47 0D 0A 1A 0A')
        return isPng
      },
      async isJpg(file) {
        const len = file.size
        const start = await this.blobToString(file.slice(0, 2))
        const tail = await this.blobToString(file.slice(-2, len))
        const isJpg = (start === 'FF D8') && (tail === 'FF D9')
        return isJpg
      },
      async isImage(file) {
        // 通过文件流来判定
        return await this.isGif(file) || await this.isPng(file) || await this.isJpg(file)
      },
      createFileChunk(file, size=CHUNK_SIZE) {
        const chunks = []
        let cur = 0
        while(cur < this.file.size) {
          chunks.push({ index: cur, file: this.file.slice(cur, cur + size) })
          cur += size
        }
        return chunks
      },
      async calculateHashWorker() { // web worker 影分身
        return new Promise(resolve => {
          this.worker = new Worker('/hash.js')
          this.worker.postMessage({ chunks: this.chunks })
          this.worker.onmessage = e => {
            const { progress, hash } = e.data
            this.hashProgress = Number(progress.toFixed(2))
            if (hash) {
              resolve(hash)
            }
          }
        })
      },
      async calculateHashIdle() { //requestIdleCallback 时间切片
        const chunks = this.chunks
        return new Promise(resolve => {
          const spark = new sparkMD5.ArrayBuffer()
          let count = 0
          const appendToSpark = async file => {
            return new Promise(resolve => {
              const reader = new FileReader()
              reader.readAsArrayBuffer(file)
              reader.onload = e => {
                spark.append(e.target.result)
                resolve()
              }
            })
          }
          const workLoop = async deadline => {
            while (count < chunks.length && deadline.timeRemaining() > 1) {
              // 空闲时间，且有任务
              await appendToSpark(chunks[count].file)
              count++
              if (count < chunks.length) {
                this.hashProgress = Number(
                  ((100 * count) / chunks.length).toFixed(2)
                )
              } else {
                this.hashProgress = 100
                resolve(spark.end())
              }
            }
            window.requestIdleCallback(workLoop)
          }
          window.requestIdleCallback(workLoop)
        })
      },
      async calculteHashSample() { // 抽样hash
        // 参考的布隆过滤器，判断一个数据存在与否
        // 1个G的文件，抽样后5M以内
        // hash一样，文件不一定一样
        // hash不一样，文件一定不一样
        return new Promise(resolve => {
          const spark = new sparkMD5.ArrayBuffer()
          const reader = new FileReader()
          const file = this.file
          const size = file.size
          const offset = 2 * 1024 * 1024
          // 第一个2M，最后一个区块 数据全要
          let chunks = [file.slice(0, offset)]
          let cur = offset
          while (cur < size) {
            if (cur + offset >= size) { 
              // 最后一个区块
              chunks.push(cur, cur + offset)
            } else {
              const mid = cur + offset / 2
              const end = cur + offset
              chunks.push(file.slice(cur, cur + 2))
              chunks.push(file.slice(mid, mid + 2))
              chunks.push(file.slice(end - 2, end))
            }
            cur += offset
          }
          reader.readAsArrayBuffer(new Blob(chunks))
          reader.onload = e => {
            spark.append(e.target.result)
            this.hashProgress = 100
            resolve(spark.end())
          }
        })
      },
      async uploadFile() {
        // 判断图片格式
        // if (!await this.isImage(this.file)) {
        //   console.log('文件格式不对')
        //   return
        // } else {
        //   console.log('格式正确')
        // }

        const chunks = this.createFileChunk(this.file)
        // const hash1 = await this.calculateHashWorker() //方式一
        // const hash2 = await this.calculateHashIdle() //方式二
        const hash = await this.calculteHashSample() //方式三
        this.hash = hash
        // console.log('web work hash1', hash1)
        // console.log('切片hash2', hash2)
        // console.log('抽样hash', this.hash)

        // 请求server，文件是否上传过，如果没有，是否存在切片
        const { data: { uploaded, uploadedList } } = await this.$http.post('/checkfile', {
          hash: this.hash,
          ext: this.file.name.split('.').pop()
        })
        if (uploaded) {
          this.$message.success('秒传成功')
        }

        this.chunks = chunks.map((chunk, index) => {
          // 切片的名字 hash + index
          const name = hash + '-' + index
          return {
            hash,
            name,
            index,
            // progress: 0,
            progress: uploadedList.indexOf(name) > -1 ? 100 : 0, // 断点上传：判断每个切片进度条，已经上传为100，没有上传为0
            chunk: chunk.file
          }
        })
        console.log(this.chunks);

        await this.uploadChunks()
        
      },
      async mergeChunks() {
        this.$http.post('/mergefile', {
          ext: this.file.name.split('.').pop(),
          size: CHUNK_SIZE,
          hash: this.hash,
        })
      },
      async uploadChunks() {
        const requests = this.chunks.map((chunk, index) => {
          // 转成promise
          const form = new FormData()
          form.append('chunk', chunk.chunk)
          form.append('hash', chunk.hash)
          form.append('name', chunk.name)
          return {form, index: chunk.index, error: 0}
          // form.append('index', chunk.index)
        })
        // .map(({form, index}) => this.$http.post('/uploadfile', form, {
        //   onUploadProgress: progressEvent => {
        //     this.chunks[index].progress = Number((progressEvent.loaded * 100 / progressEvent.total).toFixed(2))
        //   }
        // }))
        // @todo 并发量控制
        // 尝试建立tcp链接过多，也会造成卡顿
        // await Promise.all(requests)

        // 异步的并发控制
        await this.sendRequest(requests)

        await this.mergeChunks()
        // const form = new FormData() 
        // form.append('name', 'file')
        // form.append('file', this.file)
        // const ret = await this.$http.post('/uploadfile', form, {
        //   onUploadProgress: progressEvent => {
        //     this.uploadProgress = Number( (progressEvent.loaded * 100 / progressEvent.total).toFixed(2) );
        //   }
        // })
        // console.log(ret)
      },
      // 上传可能报错
      // 报错之后，进度条变红，开始重试
      // 一个切片重试失败三次，整体全部终止
      async sendRequest(chunks, limit = 4) {
        // limit 并发数
        // 一个数组，限制limit
        // [task1, task2, task3]
        return new Promise((resolve, reject) => {
          const len = chunks.length
          let counter = 0
          let isStop = false
          const start = async () => {
            if (isStop) {
              return
            }
            const task = chunks.shift()
            if (task) {
              const { form, index } = task
              try {
                await this.$http.post('/uploadfile',form,{
                  onUploadProgress:progress=>{
                    // 不是整体的进度条了，而是每个区块有自己的进度条，整体的进度条需要计算
                    this.chunks[index].progress = Number(((progress.loaded/progress.total)*100).toFixed(2))
                  }
                })
                if (counter === len - 1) {
                  resolve()
                } else {
                  counter++
                  start()
                }
              } catch (error) {
                this.chunks[index].progress = -1
                if (task.error < 3) {
                  task.error++
                  chunks.unshift(task)
                  start()
                } else {
                  isStop = true
                  reject()
                }
              }
            }
          }

          while (limit > 0) {
            setTimeout(() => {
              start()
            }, Math.random() * 2000)
            limit -= 1
          }
        })
      },
      handleFileChange(e) {
        const [file] = e.target.files
        if (!file) {
          return
        }
        this.file = file
      }
    },
  }
</script>

<style lang="stylus" scoped>
  #drag 
    height 100px
    line-height 100px
    border 2px dashed #eee
    text-align center 
    vertical-align middle
    &:hover
      border 2px dashed yellowgreen
  .mt50 
    margin-top 50px
  .cube-container
    .cube
      width 14px
      height 14px
      line-height 12px
      border 1px solid black
      background #eee
      float left
      >.success
        background green
      >.uploading
        background blue
      >.error
        background red
</style>