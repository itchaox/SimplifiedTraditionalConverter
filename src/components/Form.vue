<!--
 * @Version    : v1.00
 * @Author     : itchaox
 * @Date       : 2023-09-26 15:10
 * @LastAuthor : itchaox
 * @LastTime   : 2023-11-19 02:33
 * @desc       : 
-->
<script setup>
  import { ref } from 'vue';
  import { bitable } from '@lark-base-open/js-sdk';

  import Chinese from 'chinese-s2t';

  const raw = ref('s1');
  const target = ref('s2');

  const base = bitable.base;

  const fieldId = ref();
  const recordId = ref();

  const currentValue = ref();

  bitable.base.onSelectionChange(async (event) => {
    // 获取点击的字段id和记录id
    fieldId.value = event.data.fieldId;
    recordId.value = event.data.recordId;

    const table = await base.getActiveTable();
    if (fieldId.value && recordId.value) {
      // 修改当前数据
      let data = await table.getCellValue(fieldId.value, recordId.value);
      currentValue.value = data[0].text;
    }
  });

  async function confirm() {
    const table = await base.getActiveTable();
    let newValue;

    // 简体转繁体
    if (raw.value === 's1' && target.value === 't2') {
      newValue = Chinese.s2t(currentValue.value);
      if (fieldId.value && recordId.value) {
        await table.setCellValue(fieldId.value, recordId.value, newValue);
      }
    }

    // 繁体转简体
    if (raw.value === 't1' && target.value === 's2') {
      newValue = Chinese.t2s(currentValue.value);
      if (fieldId.value && recordId.value) {
        await table.setCellValue(fieldId.value, recordId.value, newValue);
      }
    }
  }
</script>

<template>
  <div>
    <div class="tip">
      <div class="tip-text">操作说明:</div>
      <div class="tip-text">1. 选中需要转换的单元格</div>
      <div class="tip-text">2. 再选择原文和目标的格式</div>
      <div class="tip-text">3. 点击确认转换即可</div>
    </div>
    <div class="label">
      <div class="text">原文:</div>
      <el-radio-group v-model="raw">
        <el-radio-button label="s1">简体</el-radio-button>
        <el-radio-button label="t1">繁体</el-radio-button>
      </el-radio-group>
    </div>

    <div class="label">
      <div class="text">目标:</div>
      <el-radio-group v-model="target">
        <el-radio-button label="s2">简体</el-radio-button>
        <el-radio-button label="t2">繁体</el-radio-button>
      </el-radio-group>
    </div>

    <el-button
      type="primary"
      class="btn"
      @click="confirm"
      >确认转换</el-button
    >
  </div>
</template>

<style scoped>
  .tip {
    color: #8f959e;
    font-size: 12px;
    margin-bottom: 24px;
    .tip-text {
      margin-bottom: 4px;
    }
  }

  .label {
    display: flex;
    align-items: center;
    margin-bottom: 14px;

    .text {
      margin-right: 10px;
    }
  }

  .btn {
    margin-top: 14px;
  }
</style>
