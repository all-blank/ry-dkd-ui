<template>
  <div class="app-container">
    <el-form :model="queryParams" ref="queryRef" :inline="true" v-show="showSearch" label-width="68px">
      <el-form-item label="商品名称" prop="skuName">
        <el-input v-model="queryParams.skuName" placeholder="请输入商品名称" clearable @keyup.enter="handleQuery" />
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="Search" @click="handleQuery">搜索</el-button>
        <el-button icon="Refresh" @click="resetQuery">重置</el-button>
      </el-form-item>
    </el-form>

    <el-row :gutter="10" class="mb8">
      <el-col :span="1.5">
        <el-button type="primary" plain icon="Plus" @click="handleAdd" v-hasPermi="['manage:sku:add']">新增</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button type="success" plain icon="Edit" :disabled="single" @click="handleUpdate"
          v-hasPermi="['manage:sku:edit']">修改</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button type="danger" plain icon="Delete" :disabled="multiple" @click="handleDelete"
          v-hasPermi="['manage:sku:remove']">删除</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button type="warning" plain icon="Upload" @click="handleImport"
          v-hasPermi="['manage:sku:add']">导入</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button type="warning" plain icon="Download" @click="handleExport"
          v-hasPermi="['manage:sku:export']">导出</el-button>
      </el-col>
      <right-toolbar v-model:showSearch="showSearch" @queryTable="getList"></right-toolbar>
    </el-row>

    <el-table v-loading="loading" :data="skuList" @selection-change="handleSelectionChange">
      <el-table-column type="selection" width="55" align="center" />
      <el-table-column label="序号" type="index" width="50" align="center" prop="skuId" />
      <el-table-column label="商品名称" align="center" prop="skuName" />
      <el-table-column label="商品图片" align="center" prop="skuImage" width="100">
        <template #default="scope">
          <image-preview :src="scope.row.skuImage" :width="50" :height="50" />
        </template>
      </el-table-column>
      <el-table-column label="品牌" align="center" prop="brandName" />
      <el-table-column label="规格" align="center" prop="unit" />
      <el-table-column label="商品价格" align="center" prop="price">
        <template #default="scope">
          <el-tag>{{ scope.row.price / 100 }}</el-tag>
        </template>
      </el-table-column>
      <el-table-column label="商品类型" align="center" prop="classId">
        <template #default="scope">
          <div v-for="item in skuClassList" :key="item.classId">
            <span v-if="item.classId == scope.row.classId">{{ item.className }}</span>
          </div>
        </template>
      </el-table-column>
      <el-table-column label="创建时间" align="center" prop="createTime" width="180">
        <template #default="scope">
          <span>{{ parseTime(scope.row.createTime, '{y}-{m}-{d} {h}:{i}:{s}') }}</span>
        </template>
      </el-table-column>
      <el-table-column label="操作" align="center" class-name="small-padding fixed-width">
        <template #default="scope">
          <el-button link type="primary" @click="handleUpdate(scope.row)"
            v-hasPermi="['manage:sku:edit']">修改</el-button>
          <el-button link type="primary" @click="handleDelete(scope.row)"
            v-hasPermi="['manage:sku:remove']">删除</el-button>
        </template>
      </el-table-column>
    </el-table>

    <pagination v-show="total>0" :total="total" v-model:page="queryParams.pageNum" v-model:limit="queryParams.pageSize"
      @pagination="getList" />

    <!-- 添加或修改商品管理对话框 -->
    <el-dialog :title="title" v-model="open" width="500px" append-to-body>
      <el-form ref="skuRef" :model="form" :rules="rules" label-width="80px">
        <el-form-item label="商品名称" prop="skuName">
          <el-input v-model="form.skuName" placeholder="请输入商品名称" />
        </el-form-item>
        <el-form-item label="品牌" prop="brandName">
          <el-input v-model="form.brandName" placeholder="请输入品牌" />
        </el-form-item>
        <el-form-item label="商品价格" prop="price">
          <el-input-number v-model="form.price" :mix="0.01" :max="999.99" :precision="2" style="width: 200px;"
            placeholder="请输入商品价格" />
        </el-form-item>
        <el-form-item label="商品类型" prop="classId">
          <el-select v-model="form.classId" placeholder="请选择商品类型">
            <el-option v-for="item in skuClassList" :key="item.classId" :label="item.className" :value="item.classId" />
          </el-select>
        </el-form-item>
        <el-form-item label="规格" prop="unit">
          <el-input v-model="form.unit" placeholder="请输入规格(净含量)" />
        </el-form-item>
        <el-form-item label="商品图片" prop="skuImage">
          <image-upload v-model="form.skuImage" />
        </el-form-item>
      </el-form>
      <template #footer>
        <div class="dialog-footer">
          <el-button type="primary" @click="submitForm">确 定</el-button>
          <el-button @click="cancel">取 消</el-button>
        </div>
      </template>
    </el-dialog>
    <!-- 导入Excel对话框 -->
    <el-dialog :title="title" v-model="importExcelOpen" width="400px" append-to-body>
      <excel-upload :limit="1" :fileSize="1" :visible="importExcelOpen"
        @success="handleUploadSuccess"></excel-upload>
    </el-dialog>
  </div>
</template>

<script setup name="Sku">
import { listSku, getSku, delSku, addSku, updateSku } from "@/api/manage/sku";
import { listSkuClass } from "@/api/manage/skuClass";
import { loadAllParams } from "@/api/page"
import { ref } from "vue";

const { proxy } = getCurrentInstance();

const skuList = ref([]);
const open = ref(false);
const loading = ref(true);
const showSearch = ref(true);
const ids = ref([]);
const single = ref(true);
const multiple = ref(true);
const total = ref(0);
const title = ref("");

const data = reactive({
  form: {},
  queryParams: {
    pageNum: 1,
    pageSize: 10,
    skuName: null,
    classId: null,
  },
  rules: {
    skuName: [
      { required: true, message: "商品名称不能为空", trigger: "blur" }
    ],
    skuImage: [
      { required: true, message: "商品图片不能为空", trigger: "blur" }
    ],
    brandName: [
      { required: true, message: "品牌不能为空", trigger: "blur" }
    ],
    unit: [
      { required: true, message: "规格(净含量)不能为空", trigger: "blur" }
    ],
    price: [
      { required: true, message: "商品价格，单位分不能为空", trigger: "blur" }
    ],
    classId: [
      { required: true, message: "商品类型Id不能为空", trigger: "blur" }
    ],
  }
});

const { queryParams, form, rules } = toRefs(data);

/** 查询商品管理列表 */
function getList() {
  loading.value = true;
  listSku(queryParams.value).then(response => {
    skuList.value = response.rows;
    total.value = response.total;
    loading.value = false;
  });
}

// 取消按钮
function cancel() {
  open.value = false;
  reset();
}

// 表单重置
function reset() {
  form.value = {
    skuId: null,
    skuName: null,
    skuImage: null,
    brandName: null,
    unit: null,
    price: null,
    classId: null,
    isDiscount: null,
    createTime: null,
    updateTime: null
  };
  proxy.resetForm("skuRef");
}

/** 搜索按钮操作 */
function handleQuery() {
  queryParams.value.pageNum = 1;
  getList();
}

/** 重置按钮操作 */
function resetQuery() {
  proxy.resetForm("queryRef");
  handleQuery();
}

// 多选框选中数据
function handleSelectionChange(selection) {
  ids.value = selection.map(item => item.skuId);
  single.value = selection.length != 1;
  multiple.value = !selection.length;
}

/** 新增按钮操作 */
function handleAdd() {
  reset();
  open.value = true;
  title.value = "添加商品管理";
}

/** 修改按钮操作 */
function handleUpdate(row) {
  reset();
  const _skuId = row.skuId || ids.value
  getSku(_skuId).then(response => {
    form.value = response.data;
    form.value.price /= 100; // 价格单位转换
    open.value = true;
    title.value = "修改商品管理";
  });
}

/** 提交按钮 */
function submitForm() {
  proxy.$refs["skuRef"].validate(valid => {
    if (valid) {
      form.value.price *= 100; // 价格单位转换
      if (form.value.skuId != null) {
        updateSku(form.value).then(response => {
          proxy.$modal.msgSuccess("修改成功");
          open.value = false;
          getList();
        });
      } else {
        addSku(form.value).then(response => {
          proxy.$modal.msgSuccess("新增成功");
          open.value = false;
          getList();
        });
      }
    }
  });
}

/** 删除按钮操作 */
function handleDelete(row) {
  const _skuIds = row.skuId || ids.value;
  proxy.$modal.confirm('是否确认删除商品管理编号为"' + _skuIds + '"的数据项？').then(function() {
    return delSku(_skuIds);
  }).then(() => {
    getList();
    proxy.$modal.msgSuccess("删除成功");
  }).catch(() => {});
}

/** 导出按钮操作 */
function handleExport() {
  proxy.download('manage/sku/export', {
    ...queryParams.value
  }, `sku_${new Date().getTime()}.xlsx`)
    .then(() => {    
      proxy.$modal.msgSuccess("导出Excel成功");
    }).catch(() => { 
      proxy.$modal.msgSuccess("导出Excel失败");
    });
}

/** 导入按钮操作 */
const importExcelOpen = ref(false);
function handleImport() {
  importExcelOpen.value = true;
  title.value = '数据导入';
}

// 新增上传成功处理函数
const handleUploadSuccess = () => {
  importExcelOpen.value = false; // 关闭对话框
  getList();
  proxy.$modal.msgSuccess("上传成功");
};

/** 查询商品类型列表 */
const skuClassList = ref([]);
function getSkuClassList() {
  listSkuClass(loadAllParams).then(response => {
    skuClassList.value = response.rows;
  });
}

getSkuClassList();

getList();
</script>
