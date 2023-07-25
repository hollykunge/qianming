<template>
  <div class="self-show-card" v-if="visible" ref="signPaRef">
    <div class="meet-review-flex" style="margin-top: 0">
      <p class="meet-review-flex-title">签名管理</p>
      <l-icon-sign-in-close
        style="cursor: pointer"
        @click="clickCancelChange"
      ></l-icon-sign-in-close>
    </div>
    <div class="signature-img-id-box scrollbar-self" v-if="signatureImgVisible">
      <img class="signature-img" :src="signatureImgValue" />
    </div>
    <vue-esign
      v-model:bgColor="bgColor"
      v-if="mouseSign"
      style="
         {
          height: 240px !important;
          border: 1px solid var(--color-neutral-3);
          box-shadow: 0 2px 4px 0 #0000001a;
        }
      "
      ref="esign"
      :width="800"
      :isCrop="isCrop"
      :lineWidth="lineWidth"
      :lineColor="lineColor"
    />
    <div
      class="shou-img-parent"
      v-if="state.imgBase64 == '' && !mouseSign && !signatureImgVisible"
    ></div>
    <img
      v-show="!mouseSign"
      v-if="state.imgBase64 != ''"
      :src="state.imgBase64"
      :width="450"
      :height="200"
    />

    <div
      class="meet-review-flex meet-review-flex-footer"
      v-if="!signatureImgVisible"
      :class="mouseSign ? 'margin' : ''"
    >
      <a-button
        class="meet-review-flex-buttons"
        v-show="mouseSign"
        style="margin-right: 1rem; color: #000"
        @click="handleReset"
        >清空</a-button
      >
      <a-button
        class="meet-review-flex-buttons"
        v-show="!mouseSign"
        style="margin-right: 1rem; color: #000"
        @click="handOffManner"
        >鼠标签名</a-button
      >
      <a-button
        class="meet-review-flex-buttons"
        v-show="!mouseSign"
        style="margin-right: 1rem; color: #000"
        @click="handleSignInit"
        >手写板签名</a-button
      >
      <a-button
        class="meet-review-flex-buttons"
        @click="handleGenerate"
        type="primary"
        >完成</a-button
      >
    </div>
    <div
      class="meet-review-flex meet-review-flex-footer margin"
      v-if="signatureImgVisible"
    >
      <a-button
        class="meet-review-flex-buttons"
        style="margin-right: 1rem; color: #000"
        @click="clickCancelChange"
        >取消</a-button
      >
      <a-button
        class="meet-review-flex-buttons"
        @click="updateSignature"
        type="primary"
        >修改</a-button
      >
    </div>
  </div>
</template>
<script lang="ts" setup>
import { onMounted, reactive, ref } from "vue";
import vueEsign from "vue-esign";

import {
  querySign,
  saveMeetingSign,
  uploadFile,
} from "@/service/api/services/userServices";
import { useUserStore } from "@/stores/user";
import { cleraSign, getSign } from "@/utils/signature";

import { jSign } from "./signSdk/jSign.js";
// import { setTimeout } from "timers/promises";
const visible = ref(false);
const esign = ref();
const resultImg = ref("");
const getImgSignId = ref("");
const lineWidth = 6;
const lineColor = "#000000";
const bgColor = "";
const isCrop = false;
const useUserStoreMeet = useUserStore();

// 已有签名数据
const signatureImgVisible = ref(false);
const signatureImgValue = ref();
const props = defineProps<{
  userId: string;
  status?: string;
}>();
const state = reactive({
  timerStatus: false,
  imgBase64: "",
  code: 0,
  timer: "",
});
const emit = defineEmits<{
  (e: "resultImg", val: string, getImgSignId: string): void;
  (e: "changeSignatureVisible"): void;
}>();
onMounted(() => {
  InitSign();
  if (props.status) {
    querySignFn();
  }
});
// 查询当前用户签名信息
const querySignFn = async () => {
  const res = await querySign({
    userId: useUserStoreMeet.userInfo.userId,
  });
  if (res) {
    signatureImgVisible.value = true;
    signatureImgValue.value = "api/file/downloadFile?fileId=" + res;
  }
};
const clickCancelChange = () => {
  visible.value = false;
  if (props.status) {
    emit("changeSignatureVisible");
  }
};
const handleReset = () => {
  esign.value.reset();
};
const handleGenerate = () => {
  if (!mouseSign.value) {
    resultImg.value = state.imgBase64;

    let blob = base64ToArrayBuffer(resultImg.value);
    let files = new window.File([blob], "aaa.png", { type: "image/png" });
    uploadFile(
      {
        file: files,
      },
      {
        "Content-Type": "multipart/form-data",
      }
      // eslint-disable-next-line @typescript-eslint/no-explicit-any
    ).then((res: any) => {
      getImgSignId.value = res.fileId;
      emit("resultImg", resultImg.value, getImgSignId.value);
      saveMeetingSign({
        id: res.fileId,
        fileName: res.fileName,
        fileSize: res.fileSize,
        fileExt: res.fileExt || "png",
        userId: props.userId,
        fullPath: res.fullPath,
      });
    });
  } else {
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    esign.value.generate().then((res: any) => {
      resultImg.value = res;

      let blob = base64ToArrayBuffer(resultImg.value);
      let files = new window.File([blob], "aaa.png", { type: "image/png" });
      uploadFile(
        {
          file: files,
        },
        {
          "Content-Type": "multipart/form-data",
        }
        // eslint-disable-next-line @typescript-eslint/no-explicit-any
      ).then((res: any) => {
        getImgSignId.value = res.fileId;
        emit("resultImg", resultImg.value, getImgSignId.value);
        saveMeetingSign({
          id: res.fileId,
          fileName: res.fileName,
          fileSize: res.fileSize,
          fileExt: res.fileExt || "png",
          userId: props.userId,
          fullPath: res.fullPath,
        });
      });
    });
  }

  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  // esign.value.generate().then((res: any) => {

  // });
};
function base64ToArrayBuffer(dataURI: string) {
  var mimeString = dataURI.split(",")[0].split(":")[1].split(";")[0]; // mime类型
  var byteString = atob(dataURI.split(",")[1]); //base64 解码
  var arrayBuffer = new ArrayBuffer(byteString.length); //创建ArrayBuffer
  var intArray = new Uint8Array(arrayBuffer); //创建视图
  for (var i = 0; i < byteString.length; i++) {
    intArray[i] = byteString.charCodeAt(i);
  }
  return new Blob([intArray], { type: mimeString }); // 转成 blob
}
//初始化签字版
function handleSignInit() {
  // onenSign();
  // state.timer = setInterval(handleGetSign, 1000);
  // state.timerStatus = true;

  BeginSign(signPaRef.value);
}
// 获取签字版签名 （旧版）
// function handleGetSign() {
//   // onenSign()
//   if (state.timerStatus) {
//     getSign((temp: { msgID: number; message: string }) => {
//       state.code = temp.msgID;
//       if (temp.msgID == 0) {
//         state.imgBase64 = temp.message;
//         state.timerStatus = false;
//         clearInterval(state.timer);
//         cleraSign();
//         return;
//       }
//     });
//   }
// }
// 修改签名
const updateSignature = () => {
  signatureImgVisible.value = false;
};
//打开弹框
const handleClick = () => {
  visible.value = true;
};
// 切换鼠标签名
const mouseSign = ref(false);
const handOffManner = () => {
  mouseSign.value = true;
};

/**
 * 新签名
 */
//创建一个新的插件对象
var WebSign = new jSign();
//当前接收签名的img元素
const signPaRef = ref();
//用户使用笔点击了“确认”按钮
WebSign.onConfirm = function () {
  console.log("Event:onSignConfirm");
  GetSignBase64(600, 320, true);
};
const InitSign = () => {
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  WebSign.Init((status: any) => {
    if (status) {
      //设置画笔颜色为红色
      WebSign.SetPenColor(0, 0, 0, null);
      //设置笔大小范围1-5
      WebSign.SetPenSize(1, 5, null);
    } else {
      console.log("WebSign connect fail!");
    }
  });
};
// 新手写板签名
// eslint-disable-next-line @typescript-eslint/no-explicit-any
function BeginSign(ele: any) {
  // state.imgBase64 = ele;
  ele.src = "";
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  WebSign.BeginSign((status: any) => {
    console.log("WebSign.BeginSign:" + (status ? "ok" : "fail"));
  });
}

function GetSignBase64(w, h, transparent) {
  if (WebSign) {
    WebSign.GetSignBase64(w, h, transparent, function (status, args) {
      console.log("GetSignBase64:" + status.toString());
      if (status) {
        var base64 = args[0];
        if (base64 === "") {
          return null;
        }
        var img = "data:image/png;base64," + base64;
        state.imgBase64 = img;
        //完成一次签名
        EndSign();
      } else {
        state.imgBase64 = "";
      }
    });
  }
}
//结束签名
function EndSign() {
  if (WebSign) {
    WebSign.EndSign(function (status, args) {
      console.log("EndSign:" + status.toString());
    });
  }
}

defineExpose({
  handleClick,
  clickCancelChange,
});
</script>
<style lang="less" scoped>
.self-show-card {
  position: absolute;
  z-index: 9999;
  top: 60px;
  left: 50%;
  width: 500px;
  height: 300px;
  box-sizing: border-box;
  padding: 20px;
  border: 1px solid #e5e6eb;
  background: #fff;
  box-shadow: 0 4px 10px 0 #0000001a;
  transform: translateX(-40%);

  .self-show-card-header {
    display: flex;
    align-items: center;

    .self-show-card-header-title {
      color: #1d2129;
      font-size: 16px;
      font-weight: 600;
      line-height: 24px;
      text-shadow: 0 4px 10px #0000001a;
    }
  }

  .self-show-card-footer {
    margin-top: 56px;
    text-align: right;

    .cancel-btn {
      height: 28px;
      margin-right: 20px;
      background: #f2f3f5;
      border-radius: 6px;
      box-shadow: 0 4px 10px 0 #0000001a;
      color: #4e5969;
      font-size: 14px;
      font-weight: 500;
      text-align: center;
      text-shadow: 0 4px 10px #0000001a;
    }

    .ok-btn {
      height: 28px;
      background: #165dff;
      border-radius: 6px;
      box-shadow: 0 4px 10px 0 #0000001a;
      color: #fff;
      font-size: 14px;
      font-weight: 500;
      text-align: center;
      text-shadow: 0 4px 10px #0000001a;
    }
  }
}

.meet-review-flex {
  display: flex;
  align-items: center;
  justify-content: space-between;
  // margin: 1rem 0;

  .meet-review-flex-title {
    color: #1c1f23;
    font-size: 18px;
    font-weight: 600;
  }

  .meet-review-flex-button {
    width: 60px;
    height: 24px;
    margin-right: 1rem;
    background: #0077fa;
    border-radius: 6px;
  }

  .meet-review-flex-buttons {
    // width: 54px;
    height: 32px;
    border-radius: 3px;
    color: #fff;
    font-size: 12px;
    font-weight: 400;
    line-height: 20px;
    text-align: center;
  }
}

.meet-review-flex.meet-review-flex-footer {
  justify-content: flex-end;
  margin-top: 6px;
}
.margin.meet-review-flex-footer {
  margin-top: 10px;
}

.signature-img-id-box {
  width: 450px;
  height: 200px;
  border: 1px solid var(--color-neutral-3);
  box-shadow: 0 2px 4px 0 #0000001a;
  overflow: auto;

  .signature-img {
    display: block;
    width: 100%;
    margin: auto;
  }
}
.shou-img-parent {
  width: 450px;
  height: 200px;
  background: rgba(46, 50, 56, 0.05);
}
</style>
