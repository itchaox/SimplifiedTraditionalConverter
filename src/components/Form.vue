<!--
 * @Version    : v1.00
 * @Author     : itchaox
 * @Date       : 2023-09-26 15:10
 * @LastAuthor : itchaox
 * @LastTime   : 2023-11-23 07:14
 * @desc       : 
-->
<script setup>
  import { onMounted, watch, ref, watchEffect, nextTick } from 'vue';
  import { bitable } from '@lark-base-open/js-sdk';

  import Chinese from 'chinese-s2t';

  // 目标格式 s 简体; t 繁体
  const target = ref('t');

  // 选择模式 cell 单元格; field 字段; database 数据表
  const selectModel = ref('cell');

  const databaseList = ref();
  const databaseId = ref();
  const viewList = ref();
  const viewId = ref();
  const fieldList = ref();
  const fieldId = ref();

  const isLoading = ref(false);

  const base = bitable.base;

  // 当前点击字段id
  const currentFieldId = ref();
  const recordId = ref();

  const currentValue = ref();

  onMounted(async () => {
    databaseList.value = await base.getTableMetaList();
  });

  // 切换数据表, 默认选择第一个视图
  async function databaseChange() {
    if (selectModel.value === 'field') {
      const table = await base.getTable(databaseId.value);
      viewList.value = await table.getViewMetaList();
      viewId.value = viewList.value[0]?.id;
    }
  }

  // 根据视图列表获取字段列表
  watch(viewId, async (newValue, oldValue) => {
    const table = await base.getTable(databaseId.value);
    const view = await table.getViewById(newValue);
    const _list = await view.getFieldMetaList();

    // 只展示文本相关字段
    fieldList.value = _list.filter((item) => item.type === 1);
  });

  // 切换选择模式时,重置选择
  watch(selectModel, async (newValue, oldValue) => {
    if (newValue === 'cell') return;
    // 单列和数据表模式，默认选中当前数据表和当前视图

    const selection = await base.getSelection();
    databaseId.value = selection.tableId;

    if (newValue === 'field') {
      fieldId.value = '';
      fieldList.value = [];

      const table = await base.getTable(databaseId.value);
      viewList.value = await table.getViewMetaList();
      viewId.value = selection.viewId;
    }
  });

  base.onSelectionChange(async (event) => {
    // 获取点击的字段id和记录id
    currentFieldId.value = event.data.fieldId;
    recordId.value = event.data.recordId;

    const table = await base.getActiveTable();
    if (currentFieldId.value && recordId.value) {
      // 修改当前数据
      let data = await table.getCellValue(currentFieldId.value, recordId.value);
      console.log('🚀  data:', data);
      if (data && data[0].text !== currentValue.value) {
        currentValue.value = data[0].text;
      }
    }
  });

  async function confirm() {
    isLoading.value = true;
    if (selectModel.value === 'cell') {
      await cellModel();
    } else if (selectModel.value === 'field') {
      await fieldModel();
    } else {
      await databaseModel();
    }
    isLoading.value = false;
  }

  async function cellModel() {
    const table = await base.getActiveTable();
    let newValue;

    // 简体转繁体
    if (target.value === 't') {
      newValue = Chinese.s2t(currentValue.value);
      if (currentFieldId.value && recordId.value) {
        await table.setCellValue(currentFieldId.value, recordId.value, newValue);
      }
    }

    // 繁体转简体
    if (target.value === 's') {
      newValue = Chinese.t2s(currentValue.value);
      if (currentFieldId.value && recordId.value) {
        await table.setCellValue(currentFieldId.value, recordId.value, newValue);
      }
    }
  }

  async function fieldModel() {
    ElMessage({
      message: '开始转换数据~',
      type: 'success',
    });

    const table = await bitable.base.getTable(databaseId.value);
    const recordList = await table.getRecordList();
    const recordIds = await table.getRecordIdList(); // 获取所有记录 id

    let _list = [];

    for (const record of recordList) {
      const id = record.id;
      // 获取索引
      const index = recordList.recordIdList.findIndex((iId) => iId === id);

      // FIXME 用这个api获取 cell，性能很差
      // const cell = await record.getCellByField(fieldId.value);

      const field = await table.getFieldById(fieldId.value);
      const cell = await field.getCell(recordIds[index]);
      const val = await cell.getValue();
      // const val = await cell.val;

      if (!val) continue;

      let newValue;

      // 简体转繁体
      if (target.value === 't') {
        newValue = Chinese.s2t(val[0]?.text);
      }

      // 繁体转简体
      if (target.value === 's') {
        newValue = Chinese.t2s(val[0]?.text);
      }

      // FIXME 处理数据
      _list.push({
        recordId: recordIds[index],
        fields: {
          [fieldId.value]: newValue,
        },
      });
    }

    // FIXME 此处一次性全部替换
    await table.setRecords(_list);

    ElMessage({
      message: '数据转换结束!',
      type: 'success',
    });
  }

  async function databaseModel() {
    ElMessage({
      message: '开始转换数据~',
      type: 'success',
    });

    const table = await bitable.base.getTable(databaseId.value);
    const _fieldList = await table.getFieldMetaList();
    const recordList = await table.getRecordList();
    const recordIds = await table.getRecordIdList(); // 获取所有记录 id

    const filterFieldList = _fieldList.filter((item) => item.type === 1);

    for (const item of filterFieldList) {
      let _list = [];
      for (const record of recordList) {
        const id = record.id;
        // 获取索引
        const index = recordList.recordIdList.findIndex((iId) => iId === id);

        // 只遍历文本列
        const field = await table.getFieldById(item.id);
        const cell = await field.getCell(recordIds[index]);
        const val = await cell.getValue();

        if (val) {
          let newValue;

          // 简体转繁体
          if (target.value === 't') {
            newValue = Chinese.s2t(val[0]?.text);
          }

          // 繁体转简体
          if (target.value === 's') {
            newValue = Chinese.t2s(val[0]?.text);
          }

          // FIXME 处理数据
          _list.push({
            recordId: recordIds[index],
            fields: {
              [item.id]: newValue,
            },
          });
        }
      }

      // FIXME 此处一次性全部替换
      await table.setRecords(_list);
    }

    ElMessage({
      message: '数据转换结束!',
      type: 'success',
    });
  }
</script>

<template>
  <div>
    <div class="tip">
      <div class="tip-text title">操作步骤:</div>

      <div class="tip-text">1. 选择目标格式</div>
      <div
        class="tip-text"
        v-if="selectModel === 'cell'"
      >
        2. 选择需要转换的单元格
      </div>
      <div
        class="tip-text"
        v-else-if="selectModel === 'field'"
      >
        2. 选择顺序: 数据表 -> 视图 -> 字段
      </div>
      <div
        class="tip-text"
        v-else-if="selectModel === 'database'"
      >
        2. 选择需要转换的数据表
      </div>
      <div class="tip-text">3. 点击[确认转换]按钮即可</div>
    </div>
  </div>

  <div class="label">
    <div class="text">目标格式:</div>
    <el-radio-group v-model="target">
      <el-radio-button label="s">简体</el-radio-button>
      <el-radio-button label="t">繁体</el-radio-button>
    </el-radio-group>
  </div>

  <div class="label">
    <div class="text">选择模式:</div>
    <el-radio-group v-model="selectModel">
      <el-radio-button label="cell">单元格</el-radio-button>
      <el-radio-button label="field">字段</el-radio-button>
      <el-radio-button label="database">数据表</el-radio-button>
    </el-radio-group>
  </div>

  <div
    class="label"
    v-if="selectModel !== 'cell'"
  >
    <div class="text">数据表:</div>
    <el-select
      v-model="databaseId"
      placeholder="请选择数据表"
      @change="databaseChange"
    >
      <el-option
        v-for="item in databaseList"
        :key="item.id"
        :label="item.name"
        :value="item.id"
      />
    </el-select>
  </div>

  <div
    class="label"
    v-if="selectModel === 'field'"
  >
    <div class="text">视图:</div>
    <el-select
      v-model="viewId"
      placeholder="请选择视图"
    >
      <el-option
        v-for="item in viewList"
        :key="item.id"
        :label="item.name"
        :value="item.id"
      />
    </el-select>
  </div>
  <div
    class="label"
    v-if="selectModel === 'field'"
  >
    <div class="text">字段:</div>
    <el-select
      v-model="fieldId"
      placeholder="请选择字段"
    >
      <el-option
        v-for="item in fieldList"
        :key="item.id"
        :label="item.name"
        :value="item.id"
      />
    </el-select>
  </div>

  <el-button
    type="primary"
    class="btn"
    @click="confirm"
    :loading="isLoading"
    >确认转换</el-button
  >
</template>

<style scoped>
  .tip {
    color: #8f959e;
    font-size: 12px;
    margin-bottom: 24px;
    .tip-text {
      margin-bottom: 4px;
    }

    .title {
      font-size: 14px;
      margin-bottom: 8px;
    }
  }

  .label {
    display: flex;
    align-items: center;
    margin-bottom: 20px;

    .text {
      width: 70px;
      margin-right: 10px;
      white-space: nowrap;
    }
  }

  .btn {
    margin-top: 14px;
  }
</style>
