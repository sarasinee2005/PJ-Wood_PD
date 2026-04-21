<template>
  <b-card no-body class="p-2 shadow-none border">
    <div class="no-print">
      <div class="d-flex justify-content-between align-items-center mb-2">
        <h3 class="mb-0 text-primary font-weight-bolder">ระบบคำนวณการผลิตชิ้นส่วน</h3>
        <b-button variant="flat-primary" size="sm" @click="fetchData(true)" :disabled="loading">
          <span v-if="loading">⏳ กำลังโหลด...</span>
          <span v-else>🔄 รีเฟรชข้อมูล</span>
        </b-button>
      </div>

      <b-row class="mb-2">
        <b-col md="9">
          <b-form-group label="🔍 ค้นหาและเลือกรายการ (เรียงตามชื่อสินค้า):">
            <v-select
              v-model="selectedPlan"
              :options="allPlans"
              label="display"
              placeholder="ค้นหาPO,FG code หรือชื่อสินค้า"
              @input="calculateBOM"
              class="bg-white shadow-sm"
            >
              <template #no-options="{ search }">
                {{ search.length ? 'ไม่พบข้อมูล "' + search + '"' : 'กรุณาพิมพ์เพื่อค้นหา' }}
              </template>
            </v-select>
          </b-form-group>
        </b-col>
        
        <b-col md="3">
          <div class="target-card-fixed shadow-sm border-primary">
            <div class="target-label">เป้าหมายผลิต </div>
            <div class="target-value text-primary">
              {{ targetQty.toLocaleString() }} <small>ตัว</small>
            </div>
          </div>
        </b-col>
      </b-row>

      <b-button variant="dark" block class="mb-3 font-weight-bold shadow-sm" @click="printOrder">
        🖨️ พิมพ์ใบสั่งผลิตชิ้นส่วน (A4)
      </b-button>

      <div class="table-responsive">
        <table class="table-bom">
          <thead>
            <tr>
              <th>รหัส (Part Code)</th>
              <th style="width: 30%">รายการชิ้นส่วน</th>
              <th class="text-center">ต่อตัว</th>
              <th class="text-center">ต้องใช้</th>
              <th class="text-center bg-gray">สต็อก </th>
              <th class="bg-orange text-white text-center">ผลิตเพิ่ม</th>
              <th class="bg-danger text-white text-center">ค้างผลิต</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(row, index) in tableData" :key="index">
              <td class="text-center font-weight-bold">{{ row.partCode }}</td>
              <td class="pl-1">{{ row.itemName }}</td>
              <td class="text-center">{{ row.perUnit }}</td>
              <td class="text-center text-primary font-weight-bold">{{ row.totalNeed.toLocaleString() }}</td>
              <td class="text-center bg-gray-light">{{ row.stock.toLocaleString() }}</td>
              <td class="text-center bg-orange-light text-orange font-weight-bolder">
                {{ row.toProduce.toLocaleString() }}
              </td>
              <td class="text-center bg-danger-light text-danger font-weight-bold">
                {{ row.toProduce.toLocaleString() }}
              </td>
            </tr>
            <tr v-if="selectedPlan && tableData.length === 0">
              <td colspan="7" class="text-center py-5">❌ ไม่พบข้อมูลชิ้นส่วนประกอบสำหรับ SAP นี้</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <div id="print-isolation-layer" class="print-only">
      <h1 class="text-center font-weight-bold mb-4">ใบสั่งผลิตชิ้นส่วน (Production Order)</h1>
      <div class="print-header-info">
        <div>
          <p><strong>Project Order:</strong> {{ selectedPlan ? selectedPlan.po : '-' }}</p>
          <p><strong>รหัสสินค้า (fg code):</strong> {{ selectedPlan ? selectedPlan.sap : '-' }}</p>
          <p><strong>รายการ:</strong> {{ selectedPlan ? selectedPlan.name : '-' }}</p>
        </div>
        <div class="text-right">
          <p><strong>วันที่พิมพ์:</strong> {{ new Date().toLocaleDateString('th-TH') }}</p>
          <p><strong>สี (Color):</strong> {{ selectedPlan ? selectedPlan.color : '-' }}</p>
        </div>
      </div>
      <div class="print-target-banner mb-4">
        เป้าหมายผลิตทั้งหมด: {{ targetQty.toLocaleString() }} ตัว
      </div>
      <table class="table-print">
        <thead>
          <tr>
            <th>รหัส (Part Code)</th>
            <th style="width: 30%">รายการ (Item Name)</th>
            <th>ต่อตัว</th>
            <th>ต้องผลิต</th>
            <th>สต็อก</th>
            <th class="bg-print-gray">ผลิตเพิ่ม</th>
            <th class="bg-print-gray">ค้างผลิต</th>
            <th>เบิกสี</th>
            <th>ของเสีย</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(row, index) in tableData" :key="index">
            <td class="text-center font-weight-bold">{{ row.partCode }}</td>
            <td class="pl-2">{{ row.itemName }}</td>
            <td class="text-center">{{ row.perUnit }}</td>
            <td class="text-center">{{ row.totalNeed.toLocaleString() }}</td>
            <td class="text-center">{{ row.stock.toLocaleString() }}</td>
            <td class="text-center font-weight-bold">{{ row.toProduce.toLocaleString() }}</td>
            <td class="text-center font-weight-bold">{{ row.toProduce.toLocaleString() }}</td>
            <td class="text-center"></td>
            <td class="text-center"></td>
          </tr>
        </tbody>
      </table>
      <div class="print-footer mt-5">
         <p>ฝ่ายผลิต: .................................................................................................................................................</p>
      </div>
    </div>
  </b-card>
</template>

<script>
import vSelect from 'vue-select'
import axios from 'axios'
import { BRow, BCol, BButton, BFormGroup, BCard } from 'bootstrap-vue'

export default {
  components: { vSelect, BRow, BCol, BButton, BFormGroup, BCard },
  data() {
    return {
      loading: false,
      allPlans: [],
      stockMap: {},
      bomDatabase: {},
      selectedPlan: null,
      targetQty: 0,
      tableData: []
    }
  },
  mounted() { this.fetchData() },
  methods: {
    async fetchData(force = false) {
      this.loading = true
      try {
        const urlPlan = "https://docs.google.com/spreadsheets/d/e/2PACX-1vQUfe59fFqNK3a6yzya8ZB4W_2dqqBg0c_06ZpH3QQ65J53FfxoEO5iPFW9LEksMEexwLOx0XTb5UcF/pub?output=csv"
        const urlStock = "https://docs.google.com/spreadsheets/d/e/2PACX-1vShU_0KepJ0kBsIYP6zPhKVuZEE4TbCDyvx81yzPMBbdtO3SJp1kf1bRc4hbFfs4ErULD0oDyK39CkE/pub?output=csv"
        const urlBOM = "https://docs.google.com/spreadsheets/d/e/2PACX-1vT-sDov5Hl1nWrRBz9jCiiBeatitpYjZp0fqPYngNmvdY1teoDLQrhPBNkMZATc-rktKV37X2Q1qzkm/pub?output=csv"
        
        const [resPlan, resStock, resBOM] = await Promise.all([
          axios.get(urlPlan), axios.get(urlStock), axios.get(urlBOM)
        ])
        
        // 1. Map Stock
        const sLines = resStock.data.split(/\r?\n/)
        this.stockMap = {}
        sLines.forEach((line, i) => {
          if (i === 0 || !line) return
          const cols = line.split(/,(?=(?:[^"]*"[^"]*")*[^"]*$)/)
          const code = (cols[1] || "").replace(/"/g, '').trim()
          const val = (cols[7] || "0").replace(/[",]/g, '').trim()
          if (code) this.stockMap[code] = parseInt(val) || 0
        })

        // 2. Map BOM with Fill Down Logic
        const bLines = resBOM.data.split(/\r?\n/)
        const db = {}
        let lastSap = ""
        
        bLines.forEach((line, i) => {
          if (i === 0 || !line) return
          const c = line.split(/,(?=(?:[^"]*"[^"]*")*[^"]*$)/)
          let currentSap = (c[0] || "").replace(/"/g, '').trim()
          const pCode = (c[2] || "").replace(/"/g, '').trim()
          const pName = (c[3] || "").replace(/"/g, '').trim()
          const qty = parseFloat((c[5] || "0").replace(/[",]/g, '')) || 0

          if (currentSap) { lastSap = currentSap }
          
          if (lastSap && pCode) {
            if (!db[lastSap]) db[lastSap] = []
            db[lastSap].push({ code: pCode, name: pName, perUnit: qty })
          }
        })
        this.bomDatabase = db

        // 3. Map Plan and SORT BY NAME (จัดหมวดหมู่)
        const pLines = resPlan.data.split(/\r?\n/)
        const unsortedPlans = pLines.slice(1).map(line => {
          if (!line) return null
          const c = line.split(/,(?=(?:[^"]*"[^"]*")*[^"]*$)/)
          const sap = (c[3] || "").replace(/"/g, '').trim()
          if (!sap) return null
          const po = (c[2] || "").replace(/"/g, '').trim()
          const name = (c[6] || "").replace(/"/g, '').trim()
          return {
            po, sap, name, 
            color: (c[7] || "").replace(/"/g, '').trim(),
            qty: parseInt((c[8] || "0").replace(/[",]/g, '')) || 0,
            display: `[${name}] PO: ${po} | SAP: ${sap}`
          }
        }).filter(v => v !== null)

        // เรียงลำดับตามชื่อ (Alphabetical Sort) เพื่อให้สินค้าหมวดเดียวกันอยู่ติดกัน
        this.allPlans = unsortedPlans.sort((a, b) => a.name.localeCompare(b.name, 'th'))

      } catch (err) { console.error(err) } finally { this.loading = false }
    },
    calculateBOM(val) {
      if (!val) { this.tableData = []; this.targetQty = 0; return }
      this.targetQty = val.qty
      const components = this.bomDatabase[val.sap] || []
      this.tableData = components.map(item => {
        const need = item.perUnit * this.targetQty
        const stock = this.stockMap[item.code] || 0
        return {
          partCode: item.code, itemName: item.name, perUnit: item.perUnit,
          totalNeed: need, stock: stock, toProduce: Math.max(0, need - stock)
        }
      })
    },
    printOrder() { window.print() }
  }
}
</script>

<style lang="scss" scoped>
@import url('https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;600;700&display=swap');
.target-card-fixed {
  background: #f8f8ff; border: 2px solid #7367f0; border-radius: 12px; padding: 10px; text-align: center; min-height: 110px; display: flex; flex-direction: column; justify-content: center;
  .target-label { font-size: 0.8rem; font-weight: 600; color: #5e5873; }
  .target-value { font-size: 2.5rem; font-weight: 800; line-height: 1; }
}
.table-bom {
  width: 100%; border-collapse: collapse;
  th, td { border: 1px solid #dae1e7; padding: 8px; vertical-align: middle; font-size: 0.85rem; }
  th { background-color: #5d5fef; color: white; text-align: center; }
  .bg-gray-light { background-color: #f8f8f8; }
}
.print-only { display: none; }
@media print {
  .no-print, nav, aside, footer, header { display: none !important; }
  body * { visibility: hidden; }
  #print-isolation-layer, #print-isolation-layer * { visibility: visible; }
  #print-isolation-layer { display: block !important; position: absolute; left: 0; top: 0; width: 100%; padding: 0.5cm; }
  .print-header-info { display: flex; justify-content: space-between; margin-bottom: 10px; p { margin: 0; font-size: 11pt; color: black; } }
  .print-target-banner { border: 2px solid #000; background: #eee !important; text-align: center; font-size: 1.4rem; font-weight: bold; padding: 10px; -webkit-print-color-adjust: exact; }
  .table-print { width: 100%; border-collapse: collapse; th, td { border: 1.5px solid #000 !important; padding: 8px 5px; font-size: 10pt; color: black; height: 35px; } th { background: #eee !important; -webkit-print-color-adjust: exact; } }
  @page { size: A4 portrait; margin: 1cm; }
}
</style>