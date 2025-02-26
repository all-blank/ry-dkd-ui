<template>
  <div class="upload-file">
    <el-upload :action="uploadExcelUrl" :before-upload="handleBeforeUpload" :limit="limit" :on-error="handleUploadError"
      :on-exceed="handleExceed" :on-success="handleUploadSuccess" :auto-upload="false" :headers="headers"
      class="upload-file-uploader" ref="excelUploadRef" :file-list="fileList">
      <template #trigger>
        <!-- 上传按钮 -->
        <el-button type="primary">选取文件</el-button>
      </template>
      <el-button class="ml-3" type="success" @click="submitUpload">
        上传
      </el-button>
      <template #tip>
        <!-- 上传提示 -->
        <div class="el-upload__tip" v-if="showTip">
          请上传
          <template v-if="fileSize"> 大小不超过 <b style="color: #f56c6c">{{ fileSize }}MB</b> </template>
          <template v-if="fileType"> 格式为 <b style="color: #f56c6c">{{ fileType.join("/") }}</b> </template>
          的文件
        </div>
      </template>
    </el-upload>
  </div>
</template>


<script setup>
import { getToken } from "@/utils/auth";

const props = defineProps({
  modelValue: [String, Object, Array],
  // 数量限制
  limit: {
    type: Number,
    default: 5,
  },
  // 大小限制(MB)
  fileSize: {
    type: Number,
    default: 5,
  },
  // 文件类型, 例如['png', 'jpg', 'jpeg']
  fileType: {
    type: Array,
    default: () => ["xls", "xlsx"],
  },
  // 是否显示提示
  isShowTip: {
    type: Boolean,
    default: true
  },
  visible: { // 新增 visible prop用于同步对话框的打开状态
    type: Boolean,
    default: false,
  }
});

const { proxy } = getCurrentInstance();
const emit = defineEmits(['success']);
const number = ref(0);
const uploadList = ref([]);
const baseUrl = import.meta.env.VITE_APP_BASE_API;
const uploadExcelUrl = ref(import.meta.env.VITE_APP_BASE_API + "/manage/sku/import"); // 上传文件服务器地址
const headers = ref({ Authorization: "Bearer " + getToken() });
const fileList = ref([]);
const showTip = computed(
  () => props.isShowTip && (props.fileType || props.fileSize)
);


watch(() => props.modelValue, val => {
  if (val) {
    let temp = 1;
    // 首先将值转为数组
    const list = Array.isArray(val) ? val : props.modelValue.split(',');
    // 然后将数组转为对象数组
    fileList.value = list.map(item => {
      if (typeof item === "string") {
        item = { name: item, url: item };
      }
      item.uid = item.uid || new Date().getTime() + temp++;
      return item;
    });
  } else {
    fileList.value = [];
    return [];
  }
}, { deep: true, immediate: true });

// 监听 visible 的变化，当对话框打开时清空文件列表
watch(
  () => props.visible,
  (newVal) => {
    if (newVal) {
      fileList.value = []; // 清空文件列表
    }
  }
);

// 上传前校检格式和大小
function handleBeforeUpload(file) {
  // 校检文件类型
  if (props.fileType.length) {
    const fileName = file.name.split('.');
    const fileExt = fileName[fileName.length - 1];
    const isTypeOk = props.fileType.indexOf(fileExt) >= 0;
    if (!isTypeOk) {
      proxy.$modal.msgError(`文件格式不正确, 请上传${props.fileType.join("/")}格式文件!`);
      return false;
    }
  }
  // 校检文件大小
  if (props.fileSize) {
    const isLt = file.size / 1024 / 1024 < props.fileSize;
    if (!isLt) {
      proxy.$modal.msgError(`上传Excel文件大小不能超过 ${props.fileSize} MB!`);
      return false;
    }
  }
  proxy.$modal.loading("正在上传Excel文件，请稍候...");
  number.value++;
  return true;
}

/** 上传excel文件 */
const excelUploadRef = ref({});
function submitUpload() {
  excelUploadRef.value.submit();
}

// 文件个数超出
function handleExceed() {
  proxy.$modal.msgError(`上传Excel文件数量不能超过 ${props.limit} 个!`);
}

// 上传失败
function handleUploadError(err) {
  proxy.$modal.msgError("上传Excel文件失败");
}

// 上传成功回调
function handleUploadSuccess(res, file) {

  if (res.code === 200) {
    uploadList.value.push({ name: res.fileName, url: res.fileName });
    uploadedSuccessfully();
    emit('success');
  } else {
    number.value--;
    proxy.$modal.closeLoading();
    proxy.$modal.msgError(res.msg);
    proxy.$refs.fileUpload.handleRemove(file);
    uploadedSuccessfully();
  }
}


// 上传结束处理
function uploadedSuccessfully() {
  if (number.value > 0 && uploadList.value.length === number.value) {
    fileList.value = fileList.value.filter(f => f.url !== undefined).concat(uploadList.value);
    uploadList.value = [];
    number.value = 0;
    emit("update:modelValue", listToString(fileList.value));
    proxy.$modal.closeLoading();
  }
}

// 对象转成指定字符串分隔
function listToString(list, separator) {
  let strs = "";
  separator = separator || ",";
  for (let i in list) {
    if (list[i].url) {
      strs += list[i].url + separator;
    }
  }
  return strs != '' ? strs.substr(0, strs.length - 1) : '';
}



</script>

<style scoped lang="scss">
.upload-file-uploader {
  margin-bottom: 5px;
}
</style>