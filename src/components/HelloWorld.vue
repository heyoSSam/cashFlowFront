<template>
  <div class="p-6 max-w-2xl mx-auto space-y-8">
    <!-- Выпадающие списки -->
    <div class="flex gap-6">
      <!-- Вид деятельности -->
      <select v-model="option1" class="select">
        <option disabled value="">Выбери вариант 1</option>
        <option v-for="item in options1" :key="item" :value="item">{{ item }}</option>
      </select>

      <!-- Банк -->
      <select v-model="option2" class="select">
        <option disabled value="">Выбери банк</option>
        <option v-for="bank in banks" :key="bank.value" :value="bank.value">
          {{ bank.label }}
        </option>
      </select>
    </div>

    <!-- Загрузка файла -->
    <div
      class="upload-area my-4"
      @dragover.prevent
      @drop.prevent="handleDrop"
      @click="$refs.fileInput.click()"
    >
      <p v-if="!file">Перетащи PDF сюда или кликни для выбора</p>
      <p v-else>Файл выбран: {{ file.name }}</p>
      <input type="file" ref="fileInput" class="hidden" accept=".pdf" @change="handleFile" />
    </div>

    <!-- Кнопка отправки -->
    <div class="flex justify-center">
      <button @click="sendFile" :disabled="loading || !file" class="btn">
        {{ loading ? "Загрузка..." : "Отправить" }}
      </button>
    </div>

    <!-- Результат CSV -->
    <div v-if="csvData.length" class="overflow-auto border rounded-lg p-4">
      <table class="table-auto w-full text-sm border-collapse">
        <thead>
          <tr>
            <th
              v-for="(col, idx) in csvData[0]"
              :key="idx"
              class="border px-3 py-2 bg-gray-50"
            >
              {{ col }}
            </th>
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="(row, rIdx) in csvData.slice(1)"
            :key="rIdx"
            class="hover:bg-gray-50"
          >
            <td v-for="(cell, cIdx) in row" :key="cIdx" class="border px-3 py-2">
              {{ cell }}
            </td>
          </tr>
        </tbody>
      </table>

      <!-- Итоги -->
      <div class="mt-4 text-right space-y-1">
        <div class="font-medium text-gray-700">
          Средний оборот по кредиту: {{ avgCredit.toLocaleString("ru-RU") }}
        </div>
        <div class="font-semibold text-lg">
          Максимальный ЕП ({{ (percentages[option1] * 100) || 0 }}%):
          {{ maxEP.toLocaleString("ru-RU") }}
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from "vue"

const option1 = ref("")
const option2 = ref("")
const options1 = ["Торговля", "Аренда", "СМР", "Услуги", "Производство", "Прочие"]

// список банков
const banks = [
  { label: "Kaspi bank", value: "kaspi" },
  { label: "Alatau city bank", value: "alatau" },
  { label: "Bank center credit", value: "bcc" }
]

// проценты для ЕП по видам деятельности
const percentages = {
  "Торговля": 0.3,
  "Аренда": 0.45,
  "СМР": 0.3,
  "Услуги": 0.4,
  "Производство": 0.3,
  "Прочие": 0.3
}

const file = ref(null)
const csvData = ref([])
const loading = ref(false)

const handleFile = (e) => {
  file.value = e.target.files[0]
}

const handleDrop = (e) => {
  file.value = e.dataTransfer.files[0]
}

// задержка
const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms))

const sendFile = async () => {
  if (!file.value) return
  loading.value = true

  try {
    const formData = new FormData()
    formData.append("file", file.value)
    formData.append("activity", option1.value) // вид деятельности
    formData.append("bank", option2.value)     // код банка

    let attempts = 0
    let success = false
    let text = ""

    while (attempts < 3 && !success) {
      try {
        const res = await fetch("http://localhost:8020/", {
          method: "POST",
          body: formData
        })

        if (!res.ok) throw new Error(`Ошибка сервера: ${res.status}`)

        text = await res.text()
        success = true
      } catch (err) {
        attempts++
        console.error(`Попытка ${attempts} не удалась:`, err)

        if (attempts < 3) {
          await delay(1000)
        }
      }
    }

    if (success) {
      csvData.value = text
        .trim()
        .split("\n")
        .map((row) => row.split(/\t|,/))
      console.log("Парсинг CSV:", csvData.value)
    } else {
      console.error("Файл не удалось загрузить после 3 попыток")
      alert("Ошибка загрузки файла. Попробуйте позже.")
    }
  } finally {
    loading.value = false
  }
}

// Средний оборот по кредиту
const avgCredit = computed(() => {
  if (!csvData.value.length) return 0
  const rows = csvData.value.slice(1)
  const creditColIdx = csvData.value[0].findIndex((col) =>
    col.trim().toLowerCase().includes("кредит")
  )
  if (creditColIdx === -1) return 0

  const credits = rows
    .map((row) => parseFloat(row[creditColIdx].replace(/\s/g, "")))
    .filter((v) => !isNaN(v))
  if (!credits.length) return 0

  return credits.reduce((a, b) => a + b, 0) / credits.length
})

// Максимальный ЕП по кредиту
const maxEP = computed(() => {
  if (!option1.value || !avgCredit.value) return 0
  const percent = percentages[option1.value] ?? 0
  return avgCredit.value * percent
})
</script>

<style scoped>
.select {
  border: 1px solid #ccc;
  padding: 8px 12px;
  border-radius: 8px;
  flex: 1;
  margin: 5px;
}
.upload-area {
  border: 2px dashed #ccc;
  border-radius: 12px;
  padding: 50px;
  margin: 50px;
  text-align: center;
  cursor: pointer;
  transition: 0.3s;
}
.upload-area:hover {
  border-color: #42b883;
  background: #f6fffa;
}
.btn {
  background: #42b883;
  color: white;
  padding: 12px 20px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-weight: 500;
}
.btn:disabled {
  background: #999;
  cursor: not-allowed;
}
</style>
