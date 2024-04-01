<template>
  <div id="app" style="text-align: center">
    <ul class="tool-bar" style="display: flex; justify-content: center; list-style-type: none; gap: 10px">
      <li>
        <input type="file" accept=".pdf" @change="changeFile" />
      </li>
      <li>
        <button @click="page = page > 1 ? page - 1 : page">Prev</button>
        <span>{{ page }} / {{ pages }}</span>
        <button @click="page = page < pages ? page + 1 : page">Next</button>
      </li>
      <li>
        <button @click="scale = scale > 1 ? scale - 1 : scale">-</button>
        <span>{{ Math.round((scale * 100) / 2 / 10) * 10 }}%</span>
        <button @click="scale = scale < 6 ? scale + 1 : scale">+</button>
      </li>
      <li>
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

const file = ref(null);
const { pdf, pages } = usePDF(file);

const scale = ref(2);
const page = ref(1);

function changeFile(event) {
  const selectedFile = event.target.files[0];
  if (selectedFile) {
    file.value = URL.createObjectURL(selectedFile);
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
          console.log(scale.value);
          scale.value -= 0.2;
        }
      } else if (e.deltaY < 0) {
        if (scale.value < 6) {
          console.log(scale.value);
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
        console.log(scale.value);
        scale.value -= 0.2;
      }
    } else if (e.key === "=" || e.key === "+") {
      // Ctrl + +인 경우
      e.preventDefault();
      if (scale.value < 6) {
        console.log(scale.value);
        scale.value += 0.2;
      }
    }
  }
});
function exportHTML() {
  const contentHTML = document.querySelector("html").cloneNode(true);

  // 필요없는 요소 제거
  const elementsToRemove = contentHTML.querySelectorAll(".tool-bar, script");
  elementsToRemove.forEach((element) => {
    if (element) {
      element.parentNode.removeChild(element);
    }
  });

  const filename = "exported_content.html";

  const a = document.querySelector("canvas");
  const canvasDataURL = a.toDataURL();

  const scriptElement = document.createElement("script");
  scriptElement.textContent = `
  const canvas = document.querySelector("canvas");
  const context = canvas.getContext("2d");
  const base_image = new Image();
  base_image.src = "${canvasDataURL}";
  base_image.onload = function () {
    canvas.width = base_image.width;
    canvas.height = base_image.height;
    context.drawImage(base_image, 0, 0);
  };
`;
  contentHTML.appendChild(scriptElement);
  // export
  const blob = new Blob([contentHTML.innerHTML], { type: "text/html" });

  const link = document.createElement("a");
  link.href = URL.createObjectURL(blob);
  link.download = filename;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
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
/* @import "./style/textLayer.css";
@import "./style/annotationLayer.css"; */
</style>
