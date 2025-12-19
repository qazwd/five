<template>
  <div class="background">
    <!-- 标题区域 -->
    <div class="h1-box">
      <h1 class="h1">Finish!<br>Upload Your Resume</h1>
    </div>

    <!-- 说明文本区域 -->
    <div class="text-box">
      <p>Upload your resume, the platform will help you parse and optimize, you<br>can also skip this step</p>
    </div>

    <!-- 图片区域 -->
    <div class="image">
      <img :src="require('@/assets/picture1.png')" alt="lift-Icon" class="image" />
    </div>

    <!-- 功能区域 -->
    <div class="function-box">
      <div class="h2-box">
        <h2 class="h2">Upload file</h2>
      </div>

      <!-- 文件上传区域 -->
      <div class="file-box" @click="triggerFileSelect">
        <div 
          class="file-selector" 
          id="customFileSelector"
          @dragover.prevent="handleDragOver"
          @dragleave="handleDragLeave"
          @drop.prevent="handleDrop"
        >
          <img 
            class="icon" 
            :src="require('@/assets/Vector.png')" 
            alt="up-Icon" 
          />
          <textarea 
            class="textarea" 
            id="fileInfo" 
            :value="fileDisplayText"
            readonly
            placeholder="Drag your resume file to this area, or click on the area to select the appropriate file to upload"
          ></textarea>
        </div>
      </div>

      <!-- 按钮区域 -->
      <div class="button">
        <button 
          class="last-step" 
          @click="undoLastBatch" 
          :disabled="!hasFileBatches"
        >
          Last step
        </button>
        <button class="finish" @click="showFinishMessage">Finish</button>
      </div>

      <!-- 进度条区域 -->
      <div class="load_bar">
        <img class="bar1" :src="require('@/assets/Bar.png')" alt="down-Bar_down" />
        <img class="bar2" :src="require('@/assets/Bar - Active.png')" alt="down-Bar_up" />
        <img class="round11" :src="require('@/assets/Oval.png')" alt="down-Round11" />
        <img class="round12" :src="require('@/assets/Oval-2.png')" alt="down-Round12" />
        <img class="round13" :src="require('@/assets/Check.png')" alt="down-Round13" />
        <img class="round21" :src="require('@/assets/Oval.png')" alt="down-Round21" />
        <img class="round22" :src="require('@/assets/Oval-2.png')" alt="down-Round22" />
        <img class="round23" :src="require('@/assets/Check.png')" alt="down-Round23" />
        <img class="round31" :src="require('@/assets/Oval.png')" alt="down-Round31" />
        <img class="round32" :src="require('@/assets/Oval-2.png')" alt="down-Round32" />
        <img class="round33" :src="require('@/assets/Check.png')" alt="down-Round33" />
      </div>
    </div>

    <!-- 隐藏的文件选择输入 -->
    <input 
      type="file" 
      ref="fileInput" 
      multiple 
      :accept="acceptedFileTypes" 
      class="hidden-file-input"
      @change="handleFileSelect"
    />
  </div>
</template>

<script>
export default {
  data() {
    return {
      fileBatches: [], // 存储文件批次（二维数组）
      selectedFiles: [], // 所有已选择的文件，包含完整信息
      acceptedFileTypes: '.pdf, .png, .jpg, .jpeg, .gif, .bmp, .svg, .webp, .tif, .tiff',
      fileInput: null,
      // 用于存储创建的ObjectURL，以便在适当的时候释放
      objectUrls: []
    }
  },
  computed: {
    hasFileBatches() {
      return this.fileBatches.length > 0
    },
    fileDisplayText() {
      console.log(this.selectedFiles)
      if (this.selectedFiles.length === 0) {
        return 'Drag your resume file to this area, or click on the area to select the appropriate file to upload'
      }
      
      let info = `已选择 ${this.selectedFiles.length} 个文件：\n`
      this.selectedFiles.forEach((file, index) => {
        const size = file.size > 1024 * 1024 
          ? `${(file.size / (1024 * 1024)).toFixed(2)}MB` 
          : `${(file.size / 1024).toFixed(2)}KB`
        info += `${index + 1}. ${file.name} （${size}）\n`
        info += `    路径：${file.path}\n`;
        info += `    大小：${size}\n`
      })
      return info
    }
  },
  mounted() {
    this.fileInput = this.$refs.fileInput
  },
  // beforeUnmount() {
  //   // 组件销毁前释放所有ObjectURL，避免内存泄漏
  //   this.objectUrls.forEach(url => URL.revokeObjectURL(url))
  // },
  methods: {
    // 触发文件选择
    triggerFileSelect() {
      this.fileInput.click()
    },

    // 处理文件选择
    handleFileSelect(e) {
      const newFiles = Array.from(e.target.files)
      if (newFiles.length > 0) {
        // 处理选择的文件，添加预览URL
        const filesWithInfo = newFiles.map(file => this.createFileInfo(file))
        this.addFileBatch(filesWithInfo)
        // 清空输入值，允许重复选择同一文件
        e.target.value = ''
      }
    },

    // 拖拽悬停样式处理
    handleDragOver(e) {
      e.preventDefault()
      const fileSelector = document.getElementById('customFileSelector')
      fileSelector.style.background = '#dbf1ff'
    },

    // 拖拽离开样式处理
    handleDragLeave() { 
      const fileSelector = document.getElementById('customFileSelector') 
      fileSelector.style.background = '' 
    },

    // 处理文件和文件夹拖拽
    async handleDrop(e) { 
      const fileSelector = document.getElementById('customFileSelector');
      fileSelector.style.background = '';
      
      const allValidFiles = [];
      const items = e.dataTransfer.items;
      
      if (items) {
        // 优先处理items（能识别文件夹）
        const folderPromises = [];
        for (let i = 0; i < items.length; i++) {
          const entry = items[i].webkitGetAsEntry();
          if (entry) {
            folderPromises.push(this.traverseFileTree(entry, entry.name + '/'));
          }
        }
        
        const folderFiles = await Promise.all(folderPromises);
        const flattenedFiles = [].concat(...folderFiles)
          // 过滤并创建文件信息
          .map(file => this.createFileInfo(file));
          
        allValidFiles.push(...flattenedFiles);
      } else {
        // 没有items时才处理files（兼容旧浏览器）
        const droppedFiles = Array.from(e.dataTransfer.files);
        const validFiles = droppedFiles
          .filter(file => this.isFileValid(file))
          .map(file => this.createFileInfo(file));
        
        allValidFiles.push(...validFiles);
      }

      if (allValidFiles.length > 0) {
        this.addFileBatch(allValidFiles);
      }
    },

    // 递归遍历文件夹下所有层级的文件
    traverseFileTree(item, parentPath = '') {
      return new Promise((resolve) => {
        const results = []; // 存储当前层级符合条件的文件
        let currentPath = parentPath;

        if (item.isFile) {
          // 处理文件：检查是否符合类型要求
          item.file((file) => {
            if (this.isFileValid(file)) {
              const filePath = currentPath + file.name;
              const fileWithPath = {
                       ...file,
                        path: filePath
                    };
              results.push(fileWithPath); // 符合条件则添加到结果
            }
            resolve(results);
          });
        } else if (item.isDirectory) {
          // 处理文件夹：递归遍历子项
          const dirReader = item.createReader();
          dirReader.readEntries((entries) => {
            // 收集所有子项的遍历任务（文件或文件夹）
            const entryPromises = entries.map(entry => this.traverseFileTree(entry, currentPath));
            // 等待所有子项遍历完成，合并结果
            Promise.all(entryPromises).then((allChildResults) => {
              // 展平子项结果（子文件夹可能包含多层文件）
              const flattenedResults = [].concat(...allChildResults);
              resolve(flattenedResults);
            });
          });
        } else {
          // 既不是文件也不是文件夹（如快捷方式），返回空结果
          resolve(results);
        }
      });
    },

    // 创建文件信息对象，包含预览URL
    createFileInfo(file) {
      // 创建可预览的URL
      const previewUrl = URL.createObjectURL(file);
      // 保存URL以便后续释放
      this.objectUrls.push(previewUrl);
      
      return {
        name: file.name, // 带后缀的文件名
        path: file.webkitRelativePath || file.name, // 相对路径，普通文件直接用文件名
        size: file.size, // 文件大小（字节）
        //previewUrl: previewUrl, // 可在网页预览的URL
        file: file // 原始文件对象，可能用于后续上传
      };
    },

    // 辅助方法：检查文件是否符合类型要求
    isFileValid(file) {
      // 1. 检查文件名是否有扩展名（含.）
      if (file.name.indexOf('.') === -1) {
        return false;
      }
      // 2. 提取扩展名并校验是否在允许列表中
      const fileExtension = file.name.slice(file.name.lastIndexOf('.')).toLowerCase();
      return this.acceptedFileTypes
        .split(',')
        .some(type => type.trim().toLowerCase() === fileExtension);
    },

    // 添加文件批次（去重处理）
    addFileBatch(newFiles) {
      // 去重逻辑：通过文件名、路径和大小判断是否为同一文件
      const uniqueFiles = newFiles.filter(newFile => 
       !this.selectedFiles.some(existingFile => 
          existingFile.name === newFile.name && 
          existingFile.path === newFile.path && 
          existingFile.size === newFile.size
        )
      )

      if (uniqueFiles.length > 0) {
        this.fileBatches.push(uniqueFiles)
        this.selectedFiles = [...this.selectedFiles,...uniqueFiles]
      }
    },

    // 撤销上一批文件
    undoLastBatch() {
      if (this.fileBatches.length > 0) {
        const lastBatch = this.fileBatches.pop()
        // 释放撤销文件的URL
        lastBatch.forEach(file => {
          const urlIndex = this.objectUrls.indexOf(file.previewUrl);
          if (urlIndex > -1) {
            URL.revokeObjectURL(file.previewUrl);
            this.objectUrls.splice(urlIndex, 1);
          }
        });
        
        // 更新选中文件列表
        this.selectedFiles = this.selectedFiles.filter(file => 
         !lastBatch.some(batchFile => 
            file.name === batchFile.name && 
            file.path === batchFile.path && 
            file.size === batchFile.size
          )
        )
      }
    },

    // 显示完成消息
    showFinishMessage() {
      if (this.selectedFiles.length === 0) {
        alert("请添加文件")
      } else {
        alert("提交成功！github Link：https://github.com/qazwd/Interview.git")
        // 提交后可以释放所有URL
        this.objectUrls.forEach(url => URL.revokeObjectURL(url))
        this.objectUrls = []
      }
    },

    // 可以放在文件的工具函数区域或测试用例区域
    // function testReadOnlyLengthModification() {
    //   try {
    //     // 先获取设置为只读的length属性
    //     const arr = [];
    //     Object.defineProperty(arr, 'length', { writable: false });
    //     // 尝试修改length属性，这会在严格模式下抛出错误
    //     arr.length--;
    //   } catch (error) {
    //     return error instanceof TypeError;
    //   }
    //   // 如果未抛出错误，返回false
    //   return false;
    // }
    // 后续可以通过调用该函数使用
    // console.log(testReadOnlyLengthModification()); // 输出 true
  }
}
</script>

<style scoped>
/* 隐藏文件输入框 */
.hidden-file-input {
  display: none;
}

/* 大屏（如PC端）样式 */
body {
  position: fixed;
}

/* 设置标题框样式 */
.h1-box {
  position: fixed;
  width: calc(394 / 1920 * 100vw);
  height: calc(110 / 1080 * 100vh);
  left: calc(102 / 1920 * 100vw);
  top: calc(124 / 1080 * 100vh);
  background-color: #ffffff;
  margin: 0;
  border: none;
}

.h1-box h1 {
  font-family: Inter;
  font-weight: 400;
  font-style: regular;
  font-size: min(calc(40 / 1920 * 100vw), calc(40 / 1080 * 100vh));
  line-height: calc(55 / 1080 * 100vh);
  letter-spacing: 0%;
  vertical-align: middle;
  text-transform: Title case;
  color: #000000;
}

/* 文本框样式 */
.text-box {
  position: fixed;
  width: calc(696 / 1920 * 100vw);
  height: calc(70 / 1080 * 100vh);
  top: calc(254 / 1080 * 100vh);
  left: calc(102 / 1920 * 100vw);
}

.text-box p {
  font-family: Inter;
  font-weight: 400;
  font-style: Regular;
  font-size: min(calc(20 / 1920 * 100vw), calc(20 / 1080 * 100vh));
  line-height: calc(35 / 1080 * 100vh);
  letter-spacing: 0%;
  vertical-align: Middle;
  color: #A5A5A5;
}

/* 图片样式 */
.image {
  position: fixed;
  width: calc(763 / 1920 * 100vw);
  height: calc(569 / 1080 * 100vh);
  top: calc(511 / 1080 * 100vh);
  left: 0;
}

/* 功能框样式 */
.function-box {
  position: fixed;
  width: calc(943 / 1920 * 100vw);
  height: calc(922 / 1080 * 100vh);
  top: calc(103 / 1080 * 100vh);
  left: calc(875 / 1920 * 100vw);
  border-radius: min(calc(32 / 1920 * 100vw), calc(32 / 1080 * 100vh));
  color: #ffffff;
  box-shadow: 0 min(calc(2 / 1920 * 100vw), calc(2 / 1080 * 100vh)) min(calc(20 / 1920 * 100vw), calc(20 / 1080 * 100vh)) 0 rgba(19,67,112,0.15);
}

.function-box input, img, textarea, button {position: fixed;}

/* 功能框内标题样式 */
.h2-box h2 {
  position: fixed;
  width: calc(214 / 1920 * 100vw);
  height: calc(35 / 1080 * 100vh);
  top: calc(133 / 1080 * 100vh);
  left: calc(1004 / 1920 * 100vw);
  font-family: Inter;
  font-weight: 500;
  font-style: Medium;
  font-size: min(calc(36 / 1920 * 100vw), calc(36 / 1080 * 100vh));
  line-height: calc(35 / 1080 * 100vh);
  letter-spacing: 8%;
  color: #060326;
}

/* 文件框样式 */
.file-box {
  position: fixed;
  display: flex;
  justify-content: center;
  align-items: center;
}

.file-selector {
  position: fixed;
  width: calc(685 / 1920 * 100vw);
  height: calc(420 / 1080 * 100vh);
  top: calc(250 / 1080 * 100vh);
  left: calc(1004 / 1920 * 100vw);
  border-radius: min(calc(15 / 1920 * 100vw), calc(15 / 1080 * 100vh));
  border: min(calc(1/1920*100vw), calc(1/1080*100vh)) dashed #0538BB;
  background-color: #f2faff;
  cursor: pointer;
}

.file-selector:hover {
  background-color: #dbf1ff;
}

.file-selector:active {
  background-color: #c0e5ff;
  transform: translateY(2 / 1080 * 100vh);
}

/* 框内图标样式 */
.file-selector .icon {
  width: calc(58 / 1920 * 100vw);
  height: calc(39 / 1080 * 100vh);
  top: calc(381 / 1080 * 100vh);
  left: calc(1317 / 1920 * 100vw);
  color: #1849D6;
}

/* 文本域样式 */
.textarea {
  width: calc(499 / 1920 * 100vw);
  height: calc(90 / 1080 * 100vh) !important;
  top: calc(434 / 1080 * 100vh);
  left: calc(1097 / 1920 * 100vw);
  font-family: Inter;
  font-weight: 400;
  font-style: Regular;
  font-size: min(calc(16 / 1920 * 100vw), calc(16 / 1080 * 100vh));
  line-height: calc(32 / 1080 * 100vh);
  letter-spacing: 0%;
  text-align: center;
  vertical-align: middle;
  color: #A5A5A5 !important;
  border: none;
  resize: none;
  background-color: transparent;
  white-space: pre-wrap;
}

.textarea:focus {
  outline: none;
}

#fileInfo {
  height: 150px;
  overflow-y: auto;
  overflow-x: hidden;
  outline: none;
  padding: 8px;
}

/* 按钮样式 */
.button {
  position: fixed;
}

.button button.last-step {
  width: calc(272 / 1920 * 100vw);
  height: calc(67 / 1080 * 100vh);
  top: calc(739 / 1080 * 100vh);
  left: calc(1060 / 1920 * 100vw);
  border-radius: min(calc(8 / 1920 * 100vw), calc(8 / 1080 * 100vh));
  color: #ffffff;
  background-color: #75acff;
  border: none;
  cursor: pointer;
  transition: all 0.3s ease;
}

.button button.last-step:hover {
  background-color: #5a93e6;
}

.button button.last-step:active {
  background-color: #1b5aff;
  transform: translateY(2 / 1080 * 100vh);
}

.button button.last-step:disabled {
  background-color: #75acff;
  cursor: not-allowed;
  transform: none;
}

.button button.finish {
  width: calc(272 / 1920 * 100vw);
  height: calc(67 / 1080 * 100vh);
  top: calc(739 / 1080 * 100vh);
  left: calc(1361 / 1920 * 100vw);
  border: none;
  border-radius: min(calc(8 / 1920 * 100vw), calc(8 / 1080 * 100vh));
  color: #ffffff;
  background-color: #75acff;
  cursor: pointer;
  transition: all 0.3s ease;
}

.button button.finish:hover {
  background-color: #5a93e6;
}

.button button.finish:active {
  background-color: #1b5aff;
  transform: translateY(2 / 1080 * 100vh);
}

/* 进度条样式 */
.load_bar {
  position: relative;
}

.load_bar img.bar1, .load_bar img.bar2 {
  left: calc(1175 / 1920 * 100vw);
  height: calc(4 / 1080 * 100vh);
  top: calc(910 / 1080 * 100vh);
}

.load_bar img.bar1 {
  width: calc(343 / 1920 * 100vw);
  color: #dde0e7;
}

.load_bar img.bar2 {
  width: calc(299 / 1920 * 100vw);
  border-radius: min(calc(2 / 1920 * 100vw), calc(2 / 1080 * 100vh));
  color: #85a7ff;
}

.load_bar img.round11, .load_bar img.round21, .load_bar img.round31 {
  position: fixed;
  width: calc(32 / 1920 * 100vw * 4);
  height: calc(32 / 1080 * 100vh * 4);
  top: calc(853 / 1080 * 100vh);
  color: #2a4cfe;
}

.load_bar img.round12, .load_bar img.round22, .load_bar img.round32 {
  width: calc(26 / 1920 * 100vw * 0.8);
  height: calc(26 / 1080 * 100vh * 0.8);
  top: calc(901 / 1080 * 100vh);
  color: #85a7ff;
}

.load_bar img.round13, .load_bar img.round23 {
  width: calc(10 / 1920 * 100vw);
  height: calc(7.06 / 1080 * 100vh);
  top: calc(909 / 1080 * 100vh);
  color: #2a4cfe;
}

.load_bar img.round11 {
  left: calc(1153 / 1920 * 100vw);
  box-shadow: 0 min(calc(7 / 1920 * 100vw), min(calc(64 / 1080 * 100vh))) 0 rgba(0, 0, 0, 0.07);
}

.load_bar img.round12 {
  left: calc(1206 / 1920 * 100vw);
}

.load_bar img.round13 {
  left: calc(1213 / 1920 * 100vw);
}

.load_bar img.round21 {
  left: calc(1281 / 1920 * 100vw);
  box-shadow: 0 min(calc(7 / 1920 * 100vw), calc(64 / 1080 * 100vh)) 0 rgba(0, 0, 0, 0.07);
}

.load_bar img.round22 {
  left: calc(1335 / 1920 * 100vw);
}

.load_bar img.round23 {
  left: calc(1341 / 1920 * 100vw);
}

.load_bar img.round31 {
  left: calc(1409 / 1920 * 100vw);
  box-shadow: 0 min(calc(7 / 1920 * 100vw), min(calc(64 / 1080 * 100vh))) 0 rgba(0, 0, 0, 0.07);
}

.load_bar img.round32 {
  left: calc(1462 / 1920 * 100vw);
}

.load_bar img.round33 {
  width: calc(8 / 1920 * 100vw);
  height: calc(13 / 1080 * 100vh);
  left: calc(1469 / 1920 * 100vw);
  top: calc(905 / 1080 * 100vh);
  font-family: Inter;
  font-weight: 500;
  font-style: Medium;
  font-size: min(calc(11 / 1920 * 100vw), calc(11 / 1080 * 100vh));
  line-height: calc(13 / 1080 * 100vh);
  letter-spacing: 0;
  text-transform: uppercase;
  color: #ffffff;
}

/* 移动端响应式样式 */
@media (max-width: 768px) {
  body {
    position: static;
    padding: 40px 20px 40px 20px;
  }

  .h1-box {
    position: static;
    width: 100%;
    height: auto;
    margin-bottom: 20px;
  }

  .h1-box h1 {
    font-size: 28px;
    text-align: center;
  }

  .text-box {
    position: static;
    width: 100%;
    height: auto;
    margin-bottom: 30px;
  }

  .text-box p {
    font-size: 16px;
    text-align: center;
    color: #A5A5A5;
  }

  .image {
    position: static;
    width: 100%;
    height: auto;
    margin: 0 auto 40px;
    display: block;
  }

  .function-box {
    position: static;
    width: 100%;
    height: auto;
    margin: 0 auto;
    box-shadow: 0 2px 20px 0 rgba(19,67,112,0.15);
    padding-top: 20px;
    padding-bottom: 20px;
  }

  .h2-box h2 {
    position: static;
    width: 90%;
    font-size: 24px;
    text-align: center;
    margin-bottom: 30px;
    color: #060326;
  }

  .file-box {
    position: static;
    width: 80%;
    height: 200px;
    margin: 0 auto 30px;
  }

  .file-selector {
    position: static;
    width: 100%;
    height: 100%;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
  }

  .file-selector .icon {
    position: static;
    width: 40px;
    height: auto;
    margin-bottom: 15px;
  }

  .textarea {
    position: static;
    width: 80%;
    height: auto;
    font-size: 14px;
    text-align: center;
    resize: none;
  }

  .button {
    position: static;
    display: flex;
    flex-direction: column;
    gap: 15px;
    align-items: center;
    margin: 30px 0;
  }

  .button button.last-step,
  .button button.finish {
    position: static;
    width: 80%;
    height: 50px;
    font-size: 16px;
  }

  .load_bar {
    display: none;
    padding: 30px;
  }
}
</style>