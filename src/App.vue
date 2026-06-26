<template>
  <div class="container">
    <!-- 顶部操作栏：上架商品按钮 -->
    <div class="top-bar">
      <h3>商品商城</h3>
      <button class="add-goods-btn" @click="openAddGoodsDialog">批量上架商品</button>
    </div>

    <!-- 商品选择下拉 -->
    <div class="select-goods-wrap" v-if="goodsList.length > 0">
      <span>选择查看商品：</span>
      <select v-model="currentGoodsIndex" @change="switchGoods">
        <option v-for="(item, idx) in goodsList" :key="idx" :value="idx">
          {{ item.title }}
        </option>
      </select>
      <button class="del-goods-btn" @click="deleteCurrentGoods">删除该商品</button>
    </div>
    <div class="empty-tip" v-else>暂无商品，请先点击【批量上架商品】添加</div>

    <!-- 当前商品详情区域 -->
    <template v-if="currentGoods">
      <div class="img-box">
        <img :src="currentGoods.imgList[curIndex]" alt="商品图" />
      </div>
      <div class="thumb-list">
        <img
            v-for="(item, i) in currentGoods.imgList"
            :key="i"
            :src="item"
            @click="curIndex = i"
            :class="{ active: curIndex === i }"
        />
      </div>

      <div class="info">
        <div class="price">¥{{ currentGoods.price }}</div>
        <div class="title">{{ currentGoods.title }}</div>

        <div class="spec-row">
          <span>选择规格：</span>
          <span
              class="spec-item"
              v-for="s in currentGoods.specList"
              :key="s"
              @click="selectedSpec = s"
              :class="{ active: selectedSpec === s }"
          >
            {{ s }}
          </span>
        </div>

        <div class="num-row">
          <span>购买数量：</span>
          <button @click="num--" :disabled="num <= 1">-</button>
          <input v-model.number="num" />
          <button @click="num++">+</button>
        </div>
      </div>
    </template>

    <!-- 底部操作栏 -->
    <div class="footer-bar" v-if="currentGoods">
      <div class="cart-icon-wrap" @click="openCartPanel">
        <span class="cart-icon">🛒</span>
        <span v-if="cartTotalCount > 0" class="badge">{{ cartTotalCount }}</span>
      </div>
      <button class="cart-btn" @click="addCart">加入购物车</button>
    </div>

    <!-- 加入成功提示 -->
    <div v-if="showTip" class="tip">{{ tipText }}</div>

    <!-- 购物车弹窗遮罩 -->
    <div v-if="showCartPanel" class="cart-mask" @click.self="closeCartPanel">
      <div class="cart-panel">
        <div class="cart-header">
          <h3>我的购物车</h3>
          <span class="close-btn" @click="closeCartPanel">×</span>
        </div>

        <div class="cart-list" v-if="cartList.length > 0">
          <div class="cart-item" v-for="(item, idx) in cartList" :key="idx">
            <div class="item-info">
              <div class="item-name">{{ item.goodsTitle }} {{ item.spec }}</div>
              <div class="item-price">¥{{ item.price }}</div>
            </div>
            <div class="item-num">
              <button @click="subCartNum(idx)">-</button>
              <span>{{ item.count }}</span>
              <button @click="addCartNum(idx)">+</button>
            </div>
            <div class="item-del" @click="deleteCartItem(idx)">删除</div>
          </div>
        </div>

        <div class="empty-cart" v-else>购物车暂无商品</div>

        <div class="cart-footer">
          <div class="total">合计：<span class="total-price">¥{{ totalPrice.toFixed(2) }}</span></div>
          <button class="settle-btn">去结算</button>
        </div>
      </div>
    </div>

    <!-- ========== 批量上架商品弹窗 ========== -->
    <div v-if="showAddGoodsDialog" class="dialog-mask" @click.self="closeAddGoodsDialog">
      <div class="dialog-box">
        <div class="dialog-title">
          <h3>批量上架商品</h3>
          <span class="close-btn" @click="closeAddGoodsDialog">×</span>
        </div>

        <!-- 多条商品表单组 -->
        <div class="goods-item-wrap" v-for="(goodsItem, gIdx) in batchGoodsFormList" :key="gIdx">
          <div class="item-head">
            <span>商品{{gIdx+1}}</span>
            <button v-if="batchGoodsFormList.length>1" class="del-line-btn" @click="removeGoodsLine(gIdx)">删除本条</button>
          </div>
          <div class="form-item">
            <label>商品名称</label>
            <input v-model="goodsItem.title" placeholder="输入商品名称" />
          </div>
          <div class="form-item">
            <label>商品单价</label>
            <input v-model.number="goodsItem.price" type="number" min="0" step="0.01" placeholder="输入售价" />
          </div>
          <div class="form-item">
            <label>规格（英文逗号分隔，例：39码,40码）</label>
            <input v-model="goodsItem.specStr" placeholder="多个规格用英文逗号隔开" />
          </div>
          <div class="form-item">
            <label>商品图片地址（一行一张）</label>
            <textarea v-model="goodsItem.imgStr" rows="3" placeholder="每行填写一张图片链接"></textarea>
          </div>
          <hr style="margin:12px 0;border:0;border-top:1px solid #eee;">
        </div>

        <button class="add-line-btn" @click="addGoodsLine">+ 新增一条商品</button>

        <div class="dialog-btn-group">
          <button class="cancel-btn" @click="closeAddGoodsDialog">取消</button>
          <button class="submit-btn" @click="submitBatchAddGoods">一键批量上架全部</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

// ===================== 商品库（持久化数组，支持无限新增多个商品） =====================
const goodsList = ref(JSON.parse(localStorage.getItem('goodsData')) || [])
watch(goodsList, val => {
  localStorage.setItem('goodsData', JSON.stringify(val))
}, { deep: true })

// 当前选中商品下标
const currentGoodsIndex = ref(0)
const currentGoods = computed(() => goodsList.value[currentGoodsIndex.value])

// 切换商品重置选中状态
const switchGoods = () => {
  curIndex.value = 0
  selectedSpec.value = ''
  num.value = 1
  if (currentGoods.value?.specList?.length) {
    selectedSpec.value = currentGoods.value.specList[0]
  }
}

// 删除当前商品
const deleteCurrentGoods = () => {
  if (!confirm('确定删除该商品？')) return
  goodsList.value.splice(currentGoodsIndex.value, 1)
  // 修正下标越界
  if (goodsList.value.length && currentGoodsIndex.value >= goodsList.value.length) {
    currentGoodsIndex.value = goodsList.value.length - 1
  }
}

// 商品图片轮播
const curIndex = ref(0)
let selectedSpec = ref('')
const num = ref(1)

// ===================== 批量上架商品弹窗逻辑 =====================
const showAddGoodsDialog = ref(false)
// 批量表单数组，初始默认一条商品填写位
const batchGoodsFormList = ref([
  { title: '', price: '', specStr: '', imgStr: '' }
])

// 新增一条商品填写行
const addGoodsLine = () => {
  batchGoodsFormList.value.push({ title: '', price: '', specStr: '', imgStr: '' })
}
// 删除单条填写行
const removeGoodsLine = (idx) => {
  batchGoodsFormList.value.splice(idx, 1)
}

// 打开弹窗重置表单
const openAddGoodsDialog = () => {
  showAddGoodsDialog.value = true
  batchGoodsFormList.value = [{ title: '', price: '', specStr: '', imgStr: '' }]
}
const closeAddGoodsDialog = () => showAddGoodsDialog.value = false

// 批量提交所有商品
const submitBatchAddGoods = () => {
  let successCount = 0
  for (const item of batchGoodsFormList.value) {
    const title = item.title.trim()
    const price = parseFloat(item.price)
    if (!title || isNaN(price) || price <= 0) {
      alert(`某行商品名称/价格填写非法，跳过本条`)
      continue
    }
    // 处理规格
    const specList = item.specStr.split(',')
        .map(s => s.trim())
        .filter(s => s)
    if (!specList.length) {
      alert(`商品【${title}】未填写有效规格，跳过本条`)
      continue
    }
    // 处理图片
    const imgList = item.imgStr.split('\n')
        .map(s => s.trim())
        .filter(s => s)
    if (!imgList.length) {
      alert(`商品【${title}】未填写图片链接，跳过本条`)
      continue
    }
    // 追加到商品总数组（核心：push就是新增多个，不会覆盖）
    goodsList.value.push({ title, price, specList, imgList })
    successCount++
  }

  if (successCount === 0) {
    alert('没有合法商品可上架，请检查填写内容')
    return
  }
  closeAddGoodsDialog()
  alert(`批量上架完成，成功新增${successCount}个商品！`)
  // 自动选中最后一个新商品
  currentGoodsIndex.value = goodsList.value.length - 1
  switchGoods()
}

// ===================== 购物车持久化 =====================
const showTip = ref(false)
const tipText = ref('')
const showCartPanel = ref(false)

const cartList = ref(JSON.parse(localStorage.getItem('goodsCart')) || [])
watch(cartList, val => {
  localStorage.setItem('goodsCart', JSON.stringify(val))
}, { deep: true })

const cartTotalCount = computed(() => {
  return cartList.value.reduce((sum, item) => sum + item.count, 0)
})
const totalPrice = computed(() => {
  return cartList.value.reduce((sum, item) => sum + item.price * item.count, 0)
})

// 加入购物车
const addCart = () => {
  if (!selectedSpec.value) {
    alert('请先选择商品规格')
    return
  }
  const newGoods = {
    goodsTitle: currentGoods.value.title,
    spec: selectedSpec.value,
    price: currentGoods.value.price,
    count: num.value
  }
  const existIndex = cartList.value.findIndex(
      item => item.goodsTitle === newGoods.goodsTitle && item.spec === newGoods.spec
  )
  if (existIndex > -1) {
    cartList.value[existIndex].count += newGoods.count
  } else {
    cartList.value.push(newGoods)
  }
  tipText.value = `已选中${selectedSpec.value}，数量${num.value}，加入购物车成功`
  showTip.value = true
  setTimeout(() => (showTip.value = false), 1500)
}

const openCartPanel = () => showCartPanel.value = true
const closeCartPanel = () => showCartPanel.value = false
const subCartNum = (idx) => {
  if (cartList.value[idx].count > 1) cartList.value[idx].count--
}
const addCartNum = (idx) => cartList.value[idx].count++
const deleteCartItem = (idx) => cartList.value.splice(idx, 1)

// 初始化默认选中第一个规格
if (currentGoods.value?.specList?.length) {
  selectedSpec.value = currentGoods.value.specList[0]
}
</script>

<style scoped>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
body {
  background: #f5f5f5;
  padding-bottom: 80px;
}
.container {
  max-width: 750px;
  margin: 0 auto;
  background: #fff;
  min-height: 100vh;
}
.top-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px;
  border-bottom: 1px solid #eee;
}
.add-goods-btn {
  background: #409eff;
  color: #fff;
  border: none;
  padding: 7px 14px;
  border-radius: 5px;
  cursor: pointer;
}
.select-goods-wrap {
  padding: 12px 15px;
  display: flex;
  align-items: center;
  gap: 10px;
}
.del-goods-btn {
  color: #f56c6c;
  border: 1px solid #f56c6c;
  background: #fff;
  padding: 4px 8px;
  border-radius: 4px;
  cursor: pointer;
}
.empty-tip {
  text-align: center;
  padding: 40px;
  color: #999;
}
.img-box {
  width: 100%;
  height: 420px;
  overflow: hidden;
}
.img-box img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}
.thumb-list {
  display: flex;
  gap: 8px;
  padding: 10px;
}
.thumb-list img {
  width: 70px;
  height: 70px;
  border: 2px solid transparent;
  cursor: pointer;
}
.thumb-list img.active {
  border-color: #f44336;
}
.info {
  padding: 15px;
}
.price {
  color: #f44336;
  font-size: 24px;
  font-weight: bold;
}
.title {
  font-size: 17px;
  margin: 8px 0;
}
.spec-row {
  margin: 12px 0;
}
.spec-item {
  display: inline-block;
  border: 1px solid #ccc;
  padding: 6px 14px;
  margin: 0 6px 6px 0;
  border-radius: 4px;
  cursor: pointer;
}
.spec-item.active {
  border-color: #f44336;
  color: #f44336;
}
.num-row {
  display: flex;
  align-items: center;
  margin: 15px 0;
}
.num-row button {
  width: 34px;
  height: 34px;
  border: 1px solid #ddd;
  background: #fff;
  font-size: 18px;
  cursor: pointer;
}
.num-row input {
  width: 55px;
  height: 34px;
  text-align: center;
  border: 1px solid #ddd;
  border-left: none;
  border-right: none;
}
.footer-bar {
  position: fixed;
  bottom: 0;
  left: 0;
  width: 100%;
  max-width: 750px;
  height: 65px;
  background: #fff;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  padding: 0 15px;
  border-top: 1px solid #eee;
  gap: 12px;
}
.cart-icon-wrap {
  position: relative;
  font-size: 26px;
  cursor: pointer;
  margin-right: auto;
  margin-left: 10px;
}
.badge {
  position: absolute;
  top: -5px;
  right: -5px;
  background: red;
  color: #fff;
  font-size: 12px;
  border-radius: 50%;
  width: 18px;
  height: 18px;
  text-align: center;
  line-height: 18px;
}
.cart-btn {
  background: #ff4d4f;
  color: #fff;
  border: none;
  padding: 9px 22px;
  border-radius: 20px;
  cursor: pointer;
}
.tip {
  position: fixed;
  top: 40%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(0, 0, 0, 0.7);
  color: #fff;
  padding: 12px 24px;
  border-radius: 6px;
  z-index: 99;
}
/* 购物车弹窗 */
.cart-mask {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0,0,0,0.5);
  z-index: 100;
  display: flex;
  align-items: flex-end;
}
.cart-panel {
  width: 100%;
  max-width: 750px;
  background: #fff;
  border-radius: 12px 12px 0 0;
  max-height: 70vh;
  display: flex;
  flex-direction: column;
}
.cart-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px;
  border-bottom: 1px solid #eee;
}
.close-btn {
  font-size: 24px;
  cursor: pointer;
  color: #999;
}
.cart-list {
  flex: 1;
  overflow-y: auto;
  padding: 10px 16px;
}
.cart-item {
  display: flex;
  align-items: center;
  padding: 12px 0;
  border-bottom: 1px solid #eee;
  gap: 10px;
}
.item-info {
  flex: 1;
}
.item-name {
  font-size: 15px;
}
.item-price {
  color: red;
  margin-top: 4px;
}
.item-num {
  display: flex;
  align-items: center;
  gap: 6px;
}
.item-num button {
  width: 26px;
  height: 26px;
  border: 1px solid #ddd;
  background: #fff;
}
.item-del {
  color: #f56c6c;
  font-size: 14px;
  cursor: pointer;
  padding: 0 6px;
}
.empty-cart {
  text-align: center;
  padding: 40px 0;
  color: #999;
}
.cart-footer {
  padding: 14px 16px;
  border-top: 1px solid #eee;
  display: flex;
  align-items: center;
  justify-content: space-between;
}
.total-price {
  color: red;
  font-weight: bold;
  font-size: 18px;
}
.settle-btn {
  background: #ff4d4f;
  color: #fff;
  border: none;
  padding: 8px 20px;
  border-radius: 16px;
}

/* 批量上架弹窗 */
.dialog-mask {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.5);
  z-index: 101;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
}
.dialog-box {
  width: 100%;
  max-width: 620px;
  max-height: 85vh;
  overflow-y: auto;
  background: #fff;
  border-radius: 8px;
  padding: 20px;
}
.dialog-title {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}
.goods-item-wrap {
  background: #fafafa;
  padding:12px;
  border-radius:6px;
  margin-bottom:10px;
}
.item-head {
  display:flex;
  justify-content:space-between;
  align-items:center;
  margin-bottom:8px;
  font-weight:bold;
}
.del-line-btn {
  font-size:12px;
  color:#f56c6c;
  background:#fff;
  border:1px solid #f56c6c;
  padding:2px 6px;
  border-radius:3px;
  cursor:pointer;
}
.add-line-btn {
  width:100%;
  border:1px dashed #409eff;
  background:#fff;
  color:#409eff;
  padding:8px;
  border-radius:4px;
  cursor:pointer;
  margin:8px 0;
}
.form-item {
  margin-bottom: 12px;
}
.form-item label {
  display: block;
  margin-bottom: 4px;
  font-size: 14px;
}
.form-item input,
.form-item textarea {
  width: 100%;
  border: 1px solid #ddd;
  padding: 8px 10px;
  border-radius: 4px;
}
.dialog-btn-group {
  display: flex;
  gap: 10px;
  justify-content: flex-end;
  margin-top: 15px;
}
.cancel-btn {
  border: 1px solid #ccc;
  background: #fff;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
}
.submit-btn {
  background: #409eff;
  color: #fff;
  border: none;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
}
</style>