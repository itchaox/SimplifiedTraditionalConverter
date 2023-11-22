<!--
 * @Version    : v1.00
 * @Author     : itchaox
 * @Date       : 2023-09-26 15:10
 * @LastAuthor : wangchao
 * @LastTime   : 2023-11-22 08:54
 * @desc       : 
-->
<script setup>
  import { onMounted, watch, ref, watchEffect } from "vue";
  import { bitable } from "@lark-base-open/js-sdk";

  import Chinese from "chinese-s2t";

  // 目标格式 s 简体; t 繁体
  const target = ref("t");

  // 数据范围 cell 单元格; field 单列; database 视图
  const dataRange = ref("cell");

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

  // 根据数据表获取视图列表
  watchEffect(async () => {
    const table = await bitable.base.getTable(databaseId.value);
    viewList.value = await table.getViewMetaList();
  });

  // 根据视图列表获取字段列表
  watch(viewId, async (newValue, oldValue) => {
    const table = await bitable.base.getTable(databaseId.value);
    const view = await table.getViewById(newValue);
    const _list = await view.getFieldMetaList();

    // 只展示文本相关字段
    fieldList.value = _list.filter((item) => item.type === 1);
  });

  // 切换数据纬度时,重置选择
  watch(dataRange, () => {
    databaseId.value = "";
    viewId.value = "";
    viewList.value = [];
    fieldId.value = "";
    fieldList.value = [];
  });

  bitable.base.onSelectionChange(async (event) => {
    // 获取点击的字段id和记录id
    currentFieldId.value = event.data.fieldId;
    recordId.value = event.data.recordId;

    const table = await base.getActiveTable();
    if (currentFieldId.value && recordId.value) {
      // 修改当前数据
      let data = await table.getCellValue(currentFieldId.value, recordId.value);
      console.log("🚀  data:", data);
      if (data && data[0].text !== currentValue.value) {
        currentValue.value = data[0].text;
      }
    }
  });

  async function confirm() {
    isLoading.value = true;
    if (dataRange.value === "cell") {
      await cellChange();
    } else if (dataRange.value === "field") {
      await fieldChange();
    } else {
      await databaseChange();
    }
    isLoading.value = false;
  }

  async function cellChange() {
    const table = await base.getActiveTable();
    let newValue;

    // 简体转繁体
    if (target.value === "t") {
      newValue = Chinese.s2t(currentValue.value);
      if (currentFieldId.value && recordId.value) {
        await table.setCellValue(currentFieldId.value, recordId.value, newValue);
      }
    }

    // 繁体转简体
    if (target.value === "s") {
      newValue = Chinese.t2s(currentValue.value);
      if (currentFieldId.value && recordId.value) {
        await table.setCellValue(currentFieldId.value, recordId.value, newValue);
      }
    }
  }

  async function fieldChange() {
    ElMessage({
      message: "开始转换数据~",
      type: "success",
    });

    const table = await bitable.base.getTable(databaseId.value);
    const recordList = await table.getRecordList();
    const recordIds = await table.getRecordIdList(); // 获取所有记录 id

    for (const record of recordList) {
      const id = record.id;
      // 获取索引
      const index = recordList.recordIdList.findIndex((iId) => iId === id);
      const cell = await record.getCellByField(fieldId.value);
      const val = await cell.val;
      if (!val) continue;

      let newValue;

      // 简体转繁体
      if (target.value === "t") {
        newValue = Chinese.s2t(val[0]?.text);
      }

      // 繁体转简体
      if (target.value === "s") {
        newValue = Chinese.t2s(val[0]?.text);
      }

      // 根据手机号码获取手机号码所属地
      await table.setCellValue(fieldId.value, recordIds[index], newValue);
    }

    ElMessage({
      message: "数据转换结束!",
      type: "success",
    });
  }

  async function databaseChange() {
    ElMessage({
      message: "开始转换数据~",
      type: "success",
    });

    const table = await bitable.base.getTable(databaseId.value);
    const _fieldList = await table.getFieldMetaList();
    const recordList = await table.getRecordList();
    const recordIds = await table.getRecordIdList(); // 获取所有记录 id

    for (const record of recordList) {
      const id = record.id;
      // 获取索引
      const index = recordList.recordIdList.findIndex((iId) => iId === id);

      // 只遍历文本列
      const filterFieldList = _fieldList.filter((item) => item.type === 1);
      for (const item of filterFieldList) {
        const cell = await record.getCellByField(item.id);
        const val = await cell.val;
        if (val) {
          let newValue;

          // 简体转繁体
          if (target.value === "t") {
            newValue = Chinese.s2t(val[0]?.text);
          }

          // 繁体转简体
          if (target.value === "s") {
            newValue = Chinese.t2s(val[0]?.text);
          }

          // 根据手机号码获取手机号码所属地
          await table.setCellValue(item.id, recordIds[index], newValue);
        }
      }
    }

    ElMessage({
      message: "数据转换结束!",
      type: "success",
    });
  }
</script>

<template>
  <div>
    <div class="tip">
      <div class="tip-text title">操作说明:</div>

      <div class="tip-text">1. 选择目标格式</div>
      <div
        class="tip-text"
        v-if="dataRange === 'cell'"
      >
        2. 选择需要转换的单元格
      </div>
      <div
        class="tip-text"
        v-else-if="dataRange === 'field'"
      >
        2. 选择顺序: 数据表 -> 视图 -> 字段
      </div>
      <div
        class="tip-text"
        v-else-if="dataRange === 'database'"
      >
        2. 选择需要转化的数据表
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
    <div class="text">数据纬度:</div>
    <el-radio-group v-model="dataRange">
      <el-radio-button label="cell">单元格</el-radio-button>
      <el-radio-button label="field">字段</el-radio-button>
      <el-radio-button label="database">数据表</el-radio-button>
    </el-radio-group>
  </div>

  <div
    class="label"
    v-if="dataRange !== 'cell'"
  >
    <div class="text">数据表:</div>
    <el-select
      v-model="databaseId"
      placeholder="请选择数据表"
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
    v-if="dataRange === 'field'"
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
    v-if="dataRange === 'field'"
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
