<template>
  <div id="app">
    <ul class="tool-bar">
      <li>
        <input type="file" accept=".pdf" @change="changeFile" />
      </li>
      <li>
        <button @click="page = page > 1 ? page - 1 : page">Prev</button>
        <span>{{ page }} / {{ pages }}</span>
        <button @click="page = page < pages ? page + 1 : page">Next</button>
      </li>
      <li>
        <button @click="scale = scale > 0.25 ? scale - 0.25 : scale">-</button>
        <span>{{ scale * 100 }}%</span>
        <button @click="scale = scale < 2 ? scale + 0.25 : scale">+</button>
      </li>
    </ul>
    <div class="content">
      <VuePDF :pdf="pdf" :page="page" :scale="scale" text-layer="" />
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";
import { VuePDF, usePDF } from "@tato30/vue-pdf";

const file = ref(null);
const { pdf, pages } = usePDF(file);

const scale = ref(1);
const page = ref(1);

function changeFile(event) {
  const selectedFile = event.target.files[0];
  if (selectedFile) {
    file.value = URL.createObjectURL(selectedFile);
  }
}
</script>

<style scoped></style>

<style lang="css">
#app {
  text-align: center;
}
.tool-bar {
  display: flex;
  justify-content: center;
  list-style-type: none;
  gap: 10px;
}
.content {
  width: fit-content;
  height: auto;
  margin: 0 auto;
}
@import "./style/textLayer.css";
@import "./style/annotationLayer.css";
</style>
