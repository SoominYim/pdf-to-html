<template>
  <div id="app" style="text-align: center">
    <ul class="tool-bar" style="display: flex; justify-content: center; list-style-type: none; gap: 10px">
      <li>
        <input type="file" accept=".pdf" @change="changeFile" />
      </li>
      <li v-if="isFile">
        <button @click="page = page > 1 ? page - 1 : page">Prev</button>
        <input
          type="text"
          :value="page"
          @keydown.enter="changePage"
          @focusout="resetPage"
          style="width: 50px; text-align: right"
        />
        /
        {{ pages }}
        <button @click="page = page < pages ? page + 1 : page">Next</button>
      </li>
      <li v-if="isFile">
        <button @click="scale = scale > 1.1 ? scale - 0.2 : scale">-</button>
        <span for="magnification">{{ Math.round((scale * 100) / 2 / 10) * 10 }}%</span>
        <button @click="scale = scale < 6 ? scale + 0.2 : scale">+</button>
      </li>
      <li>
        <span v-if="isFile">선택 : {{ selectedPage.map((v) => v.page) }}</span>
        <button @click="selectPage">선택</button>
      </li>
      <li v-if="isFile">
        <button @click="exportHTML">내보내기</button>
      </li>
    </ul>
    <div class="content" ref="content" :style="{ width: 'fit-content', margin: '0 auto' }">
      <VuePDF ref="vuePDFRef" :scale="scale" :pdf="pdf" :page="page" text-layer />
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import { VuePDF, usePDF } from "@tato30/vue-pdf";
import JSZip from "jszip";
import cssContent from "./style/style.js";
const file = ref(null);
const { pdf, pages } = usePDF(file);

const scale = ref(2);
const page = ref(1);
const fileName = ref("");
const isFile = ref(false);
const selectedPage = ref([]);

function changePage(e) {
  e.target.value > pages.value || e.target.value < 1 ? (e.target.value = page.value) : (page.value = +e.target.value);
}
function resetPage(e) {
  e.target.value = page.value;
}

function changeFile(event) {
  const selectedFile = event.target.files[0];
  if (selectedFile) {
    file.value = URL.createObjectURL(selectedFile);
    fileName.value = selectedFile.name.split(".").slice(0, -1).join(".");
    isFile.value = true;
  }
}

// 버려지는 br 태그 제거
onMounted(() => {
  const observer = new MutationObserver((mutationsList) => {
    mutationsList.forEach((mutation) => {
      if (mutation.addedNodes) {
        mutation.addedNodes.forEach((node) => {
          if (node.tagName === "BR") {
            node.remove();
          }
        });
      }
    });
  });

  const contentElement = document.querySelector(".content");
  observer.observe(contentElement, { childList: true, subtree: true });
});

let isCtrl = false;

document.addEventListener("keydown", function (e) {
  if (e.which === 17) {
    isCtrl = true;
  }
});

document.addEventListener("keyup", function (e) {
  if (e.which === 17) {
    isCtrl = false;
  }
});

document.addEventListener(
  "wheel",
  function (e) {
    if (isCtrl) {
      e.preventDefault();
      if (e.deltaY > 0) {
        if (scale.value > 1.1) {
          scale.value -= 0.2;
        }
      } else if (e.deltaY < 0) {
        if (scale.value < 6) {
          scale.value += 0.2;
        }
      }
    }
  },
  { passive: false }
);

document.addEventListener("keydown", function (e) {
  if (e.ctrlKey) {
    if (e.key === "-") {
      // Ctrl + -인 경우
      e.preventDefault();
      if (scale.value > 1.1) {
        scale.value -= 0.2;
      }
    } else if (e.key === "=" || e.key === "+") {
      // Ctrl + +인 경우
      e.preventDefault();
      if (scale.value < 6) {
        scale.value += 0.2;
      }
    }
  }
});

function selectPage() {
  const a = document.querySelector("canvas");
  const canvasDataURL = a.toDataURL();

  selectedPage.value.push({
    page: page.value,
    data: canvasDataURL,
  });

  console.log(selectedPage.value);
}

function exportHTML() {
  const zip = new JSZip(); // ZIP 객체 생성

  selectedPage.value.forEach((v) => {
    // 페이지 별로 HTML 복제 및 수정
    const contentHTML = document.querySelector("html").cloneNode(true);
    const elementsToRemove = contentHTML.querySelectorAll(".tool-bar, script, style");
    elementsToRemove.forEach((element) => element.parentNode.removeChild(element));

    const linkElement = document.createElement("link");
    linkElement.rel = "stylesheet";
    linkElement.href = "./css/common.css";
    contentHTML.querySelector("head").appendChild(linkElement);

    // 페이지 제목 설정
    contentHTML.querySelector("title").textContent = `${fileName.value}_${String(v.page).padStart(3, "0")}`;

    // 스크립트 직접 추가
    const scriptContent = `
            let canvas = document.querySelector("canvas");
            const context = canvas.getContext("2d");
            const base_image = new Image();
            base_image.src = "${v.data}";
            base_image.onload = function () {
                canvas.width = base_image.width;
                canvas.height = base_image.height;
                context.drawImage(base_image, 0, 0);
            };
        `;
    const scriptFileName = `${fileName.value}_${String(v.page).padStart(3, "0")}.js`;
    zip.folder("js").file(scriptFileName, scriptContent);

    const scriptElement = document.createElement("script");
    scriptElement.src = `./js/${scriptFileName}`;
    contentHTML.querySelector("body").appendChild(scriptElement);

    // Blob 생성 및 ZIP 파일에 추가
    const blob = new Blob([contentHTML.innerHTML], { type: "text/html" });
    zip.file(`${fileName.value}_${String(v.page).padStart(3, "0")}.html`, blob);
  });

  // ZIP 파일 생성 및 다운로드

  zip.folder("css").file("common.css", cssContent);

  zip
    .generateAsync({ type: "blob" }) //압축파일 생성
    .then((resZip) => {
      const url = URL.createObjectURL(resZip); //객체 URL 생성
      const aTag = document.createElement("a");

      aTag.download = fileName.value; //저장될 파일 이름
      aTag.href = url;
      aTag.click();
    });
}
</script>

<script>
export default {
  name: "app",
  data() {
    return {};
  },
  methods: {},
};
</script>
<style scoped></style>

<style lang="css">
/* @import "./style/textLayer.css"; */
@import "./style/annotationLayer.css";
</style>
