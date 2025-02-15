<template>
  <el-tabs v-model="tabActive" type="card" editable class="tabs" @tab-add="addTabs" @tab-remove="delTabs">
    <el-tab-pane v-for="(item, id) in editableTabs" :key="item" :label="item" :name="item">
      <div class="tab_item_box">
        <el-card
          style="width: 300px;margin-right: 20px;flex-shrink: 0;margin-bottom: 20px;position: relative;--el-card-padding: 0;"
          v-for="(i, ind) in metas[id]" :key="ind">
          <el-tag type="primary" style="position: absolute;top: 0;left: 0;z-index: 2;">{{ i.width }}×{{ i.height
            }}</el-tag>
          <el-image style="width: 100%; height: 250px;display: block;" :src="i.src" :zoom-rate="1.2" :max-scale="7"
            :min-scale="0.2" :preview-src-list="metasUrls[id]" show-progress :initial-index="4" fit="cover" />
          <template #footer>
            <el-form :inline="true" :model="i" style="padding-top: 15px;padding-left: 15px;">
              <el-form-item label="x:">
                <el-input v-model="i.x" placeholder="X坐标" clearable />
              </el-form-item>
              <el-form-item label="y:">
                <el-input v-model="i.y" placeholder="Y坐标" clearable />
              </el-form-item>
            </el-form>
          </template>
        </el-card>

        <el-card style="width: 300px;cursor: pointer;margin-bottom: 20px;flex-shrink: 0;" @click="chooseImg(id)">
          <div style="width: 100%;height: 250px;display: flex;align-items: center;justify-content: center;">
            <el-icon size="50px">
              <Plus />
            </el-icon>
          </div>
        </el-card>
      </div>
    </el-tab-pane>
  </el-tabs>
  <div style="display: flex;align-items: center;justify-content: center;">
    <el-button type="primary" @click="popStatus = true" :loading="loading">开始合成</el-button>
    <el-button type="primary" @click="exportImgs" style="margin-left: 100px;" :loading="loading">导出</el-button>
  </div>
  <div style="display: flex;flex-wrap: wrap;margin-top: 20px;">
    <el-image style="height: 100px;width: 100px;margin-right: 20px;margin-bottom: 20px;" :src="item" :zoom-rate="1.2"
      :max-scale="7" :min-scale="0.2" :preview-src-list="list" show-progress :initial-index="4" fit="cover"
      v-for="(item, index) in list" :key="index"></el-image>
  </div>
  <el-dialog v-model="popStatus" title="填写合成基本信息" width="500">
    <el-form>
      <el-form-item label="画板宽度(最好为第一层的宽度)">
        <el-input v-model="width" autocomplete="off" />
      </el-form-item>
      <el-form-item label="画板高度(最好为第一层的高度)">
        <el-input v-model="height" autocomplete="off" />
      </el-form-item>
      <el-form-item label="压缩比例(比例为包含100~1之间)">
        <el-input v-model="compress" autocomplete="off" />
      </el-form-item>
    </el-form>
    <template #footer>
      <div class="dialog-footer">
        <el-button @click="popStatus = false">取消</el-button>
        <el-button type="primary" @click="start">
          开始合成
        </el-button>
      </div>
    </template>
  </el-dialog>
  <canvas :style="{ width: width + 'px', height: height + 'px' }" :width="width + 'px'" ref="bill"
    :height="height + 'px'" class="bill"></canvas>
</template>

<script setup>
import JSZip from 'jszip';
import { saveAs } from 'file-saver';
import { ref } from "vue";
import { Plus } from '@element-plus/icons-vue'
import { ElMessage } from "element-plus";

let width = ref(0)
let height = ref(0)
let compress = ref(100)
let popStatus = ref(false)

let tabActive = ref(-1)

let metasUrls = ref([])
let metas = ref([])

let editableTabs = ref([])

function addTabs() {
  let name = `第${editableTabs.value.length + 1}层`
  tabActive.value = name
  editableTabs.value.push(name)
  metasUrls.value.push([])
  metas.value.push([])
}

function delTabs(e) {
  let index = editableTabs.value.indexOf(e)
  editableTabs.value.splice(index, 1)
  metasUrls.value.splice(index, 1)
  metas.value.splice(index, 1)
}

function loadImg(url) {
  return new Promise(s => {
    let img = new Image()
    img.src = url
    img.onload = e => {
      s(img)
    }
  })
}

async function chooseImg(index) {
  try {
    const handles = await window.showOpenFilePicker({
      multiple: true,
      types: [
        {
          description: "Images",
          accept: {
            "image/*": [".png", ".jpg", ".jpeg"]
          }
        }
      ]
    });
    for (const handle of handles) {
      const file = await handle.getFile();
      const imageUrl = URL.createObjectURL(file);
      const size = await loadImg(imageUrl)
      metasUrls.value[index].push(imageUrl)
      let beforeSize = metas.value[index].filter(i => i.width === size.width && i.height === size.height)[0]
      metas.value[index].push({
        width: size.width,
        height: size.height,
        src: imageUrl,
        x: beforeSize ? beforeSize.x : 0,
        y: beforeSize ? beforeSize.y : 0
      })
    }
  } catch (e) {
    console.log(e);

  }
}

// 随机组合
let allLength = 0
function calcTotalLength() {
  let l = 1
  for (let id = 0; id < metasUrls.value.length; id++) {
    l *= metasUrls.value[id].length
  }
  allLength = l
}
function getRandomElement(arr) {
  return Math.floor(Math.random() * arr.length);
}

function generateAllUniqueCombinations() {
  const selected = new Set();
  const allCombinations = [];
  while (true) {
    let combination = [];
    for (let id = 0; id < metasUrls.value.length; id++) {
      combination.push(getRandomElement(metasUrls.value[id]))
    }
    const combinationKey = combination.join(',');
    if (!selected.has(combinationKey)) {
      selected.add(combinationKey);
      allCombinations.push(combination);
    }
    // 检查是否生成了所有可能的组合
    if (selected.size === allLength) {
      break;
    }
  }
  return allCombinations;
}

// 绘制单元
let bill = ref(null)
// 绘制
async function drawImage(ctx, indexs) {
  for (let id = 0; id < indexs.length; id++) {
    let item = metas.value[id][indexs[id]]
    const res = await loadImg(item.src)
    ctx.drawImage(res, item.x, item.y)
  }
}

let list = ref([])
let loading = ref(false)
async function start() {
  if (loading.value) return
  if (!width.value) {
    ElMessage.error('请填写画板宽度')
    return
  }
  if (!height.value) {
    ElMessage.error('请填写画板高度')
    return
  }
  // 计算合成数量
  calcTotalLength()
  if (metasUrls.value.length < 2 || !metasUrls.value[1].length) {
    ElMessage.error('请选择至少2张不同层级的图片')
    return
  }
  popStatus.value = false
  loading.value = true
  list.value = []
  // 随机打散元数据并获取索引
  const metaIndexs = generateAllUniqueCombinations()
  // 清空画布
  const ctx = bill.value.getContext('2d')
  const newCanvas = document.createElement('canvas');
  const newCtx = newCanvas.getContext('2d');
  newCanvas.width = bill.value.width * compress.value / 100;
  newCanvas.height = bill.value.height * compress.value / 100;
  for (let index = 0; index < metaIndexs.length; index++) {
    ctx.clearRect(0, 0, width.value, height.value);
    const item = metaIndexs[index];
    await drawImage(ctx, item)
    if (compress.value !== 100) {
      newCtx.clearRect(0, 0, newCanvas.width, newCanvas.height);
      newCtx.drawImage(bill.value, 0, 0, newCanvas.width, newCanvas.height);
      list.value.push(newCanvas.toDataURL('image/png'))
    } else {
      list.value.push(bill.value.toDataURL('image/png'))
    }
  }
  console.log(list.value);

  loading.value = false
}

function base64ToBlob(base64) {
  const parts = base64.split(';base64,');
  const mimeType = parts[0].split(':')[1];
  const byteCharacters = atob(parts[1]);
  const byteArrays = [];
  for (let offset = 0; offset < byteCharacters.length; offset += 1024) {
    const slice = byteCharacters.slice(offset, offset + 1024);
    const byteNumbers = new Array(slice.length);
    for (let i = 0; i < slice.length; i++) {
      byteNumbers[i] = slice.charCodeAt(i);
    }
    const byteArray = new Uint8Array(byteNumbers);
    byteArrays.push(byteArray);
  }
  return new Blob(byteArrays, { type: mimeType });
}

async function exportImgs() {
  const zip = new JSZip();
  for (let i = 0; i < list.value.length; i++) {
    const blob = base64ToBlob(list.value[i]);
    const filename = `image${i + 1}.${list.value[i].split(';')[0].split('/')[1]}`; // 自动生成文件名
    zip.file(filename, blob); // 将 Blob 添加到 ZIP 中
  }
  // 生成 ZIP 文件并触发下载
  const zipContent = await zip.generateAsync({ type: 'blob' });
  saveAs(zipContent, 'images.zip');
}

</script>
<style scoped>
.tabs {
  width: 100%;
  padding-bottom: 20px;
}

.tab_item_box {
  width: 100%;
  display: flex;
  align-items: center;
  flex-wrap: nowrap;
  overflow-x: scroll;
}

.tab_item_box::-webkit-scrollbar {
  height: 3px;
}

.tab_item_box::-webkit-scrollbar-track {
  background: #ddd;
}

.tab_item_box::-webkit-scrollbar-thumb {
  background: #409eff;
}

.bill {
  position: fixed;
  left: 200vw;
}
</style>
