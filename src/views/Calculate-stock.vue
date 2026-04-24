<template>
  <b-card no-body class="p-2 shadow-none border">
    <div class="no-print">
      <div class="d-flex justify-content-between align-items-center mb-2">
        <h3 class="mb-0 text-primary font-weight-bolder">ระบบคำนวณการผลิตชิ้นส่วน</h3>
        <div class="d-flex" style="gap: 8px;">
          <b-button variant="outline-warning" size="sm" @click="clearLocalStorage">🔄 รีเซ็ตยอดที่ตัดในเครื่อง</b-button>
          <b-button variant="flat-primary" size="sm" @click="fetchData(true)" :disabled="loading">
            <span v-if="loading">⏳ กำลังโหลด...</span>
            <span v-else>🔄 รีเฟรชข้อมูล</span>
          </b-button>
        </div>
      </div>

      <b-row class="mb-3">
        <b-col md="6">
          <b-form-group>
            <template #label>
              <span class="text-primary font-weight-bold" style="font-size: 1.1rem;">
                🔍 ค้นหาด่วน (PO | SAP | รายการ | สี):
              </span>
            </template>
            <v-select
              v-model="searchTemp"
              :options="searchOptions"
              :filterable="false"
              @search="onSearch"
              label="display"
              placeholder="พิมพ์เพื่อค้นหาเร็ว..."
              @input="handleSelect"
              class="bg-white shadow-sm custom-v-select"
            >
              <template #no-options="{ search }">
                {{ search.length ? 'ไม่พบข้อมูล "' + search + '"' : 'กรุณาพิมพ์เพื่อค้นหา' }}
              </template>
            </v-select>
          </b-form-group>
        </b-col>
        
        <b-col md="3">
          <b-form-group label="🎯 เป้าหมายผลิตวันนี้:" label-class="text-danger font-weight-bold">
            <b-form-input 
              v-model.number="dailyTarget" 
              type="number" 
              :state="dailyTarget > 0 && (selectedPlan ? dailyTarget <= selectedPlan.remainingQty : true)"
              class="target-input-daily"
              @input="updateCalculation"
            />
            <small v-if="selectedPlan && dailyTarget > selectedPlan.remainingQty" class="text-danger">
              ❌ เกินยอดคงเหลือ ({{ selectedPlan.remainingQty }})
            </small>
          </b-form-group>
        </b-col>

        <b-col md="3">
          <div class="target-card-fixed shadow-sm border-primary">
            <div class="target-label">เป้าหมายคงเหลือใน PO</div>
            <div class="target-value text-primary">
              {{ selectedPlan ? selectedPlan.remainingQty.toLocaleString() : '0' }} <small>ตัว</small>
            </div>
            <div class="small text-muted" v-if="selectedPlan">จากทั้งหมด: {{ selectedPlan.qty.toLocaleString() }}</div>
          </div>
        </b-col>
      </b-row>

      <div v-if="!selectedPlan" class="plan-selection-area">
        <div class="table-responsive bg-white rounded shadow-sm border">
          <table class="table table-hover mb-0" style="table-layout: fixed;">
            <thead class="bg-light text-secondary small font-weight-bold">
              <tr>
                <th style="width: 45px;" class="text-center">#</th>
                <th style="width: 70px;">PO BATCH</th>
                <th style="width: 160px;">CODE SAP</th>
                <th style="width: 160px;">DESCRIPTION (รายการ)</th>
                <th style="width: 140px;">COLOR (สี)</th>
                <th style="width: 110px;" class="text-center">REMAINING</th>
              </tr>
              <tr class="filter-row-bg">
                <th class="py-1"></th>
                <th class="py-1"><b-form-input v-model="filterPO" size="sm" placeholder="กรอง PO..." class="filter-input-style" /></th>
                <th class="py-1"><b-form-input v-model="filterSAP" size="sm" placeholder="กรอง SAP..." class="filter-input-style" /></th>
                <th class="py-1"><b-form-input v-model="filterName" size="sm" placeholder="กรองชื่อรายการ..." class="filter-input-style" /></th>
                <th class="py-1"><b-form-input v-model="filterColor" size="sm" placeholder="กรองสี..." class="filter-input-style" /></th>
                <th class="py-1"></th>
              </tr>
            </thead>
            <tbody class="small">
              <tr v-for="(p, i) in paginatedPlans" :key="i" @click="handleSelect(p)" class="cursor-pointer plan-row">
                <td class="align-middle text-center text-muted">{{ (currentPage - 1) * perPage + (i + 1) }}</td>
                <td class="align-middle text-truncate">{{ p.po }}</td>
                <td class="align-middle font-weight-bold text-truncate">{{ p.sap }}</td>
                <td class="align-middle text-truncate">{{ p.name }}</td>
                <td class="align-middle text-truncate">{{ p.color }}</td>
                <td class="align-middle text-center">
                   <b-badge :variant="p.remainingQty > 0 ? 'light-primary' : 'light-success'">
                     {{ p.remainingQty.toLocaleString() }} / {{ p.qty.toLocaleString() }}
                   </b-badge>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
        <div class="d-flex justify-content-between align-items-center mt-3 px-1">
          <div class="text-muted small font-weight-bold">แสดงทั้งหมด {{ filteredPlans.length }} รายการ</div>
          <b-pagination v-model="currentPage" :total-rows="filteredPlans.length" :per-page="perPage" align="right" class="mb-0 pagination-purple" />
        </div>
      </div>

      <div v-else class="bom-detail-area mt-4">
        <div class="px-3 py-2 bg-light border rounded mb-3 d-flex justify-content-between align-items-center shadow-sm">
          <div>
            <span class="badge badge-primary mr-2">SELECTED</span>
            <strong>PO:</strong> {{ selectedPlan.po }} | <strong>SAP:</strong> {{ selectedPlan.sap }} | <strong>สี:</strong> {{ selectedPlan.color }} | <strong>คงเหลือ:</strong> <span class="text-danger font-weight-bold">{{ selectedPlan.remainingQty.toLocaleString() }} ตัว</span>
          </div>
          <b-button variant="outline-danger" size="sm" @click="clearSelection">ยกเลิก/เปลี่ยนรายการ</b-button>
        </div>

        <b-button 
          v-ripple.400="'rgba(255, 255, 255, 0.15)'" 
          variant="dark" 
          class="mb-3 font-weight-bold shadow-sm" 
          :disabled="dailyTarget <= 0 || dailyTarget > selectedPlan.remainingQty"
          @click="printOrder" 
          block
        >
          🖨️ พิมพ์ใบสั่งผลิตและตัดยอดคงเหลือ (A4)
        </b-button>

        <div class="table-responsive">
          <table class="table-bom">
            <thead>
              <tr>
                <th style="width: 5%">#</th>
                <th>รหัส (Part Code)</th>
                <th style="width: 35%">รายการชิ้นส่วน</th>
                <th class="text-center">ต่อตัว</th>
                <th class="text-center">ต้องใช้ (วันนี้)</th>
                <th class="text-center bg-gray">สต็อก</th>
                <th class="bg-orange text-white text-center">ผลิตเพิ่มวันนี้</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(row, index) in tableData" :key="index">
                <td class="text-center text-muted small">{{ index + 1 }}</td>
                <td class="text-center font-weight-bold">{{ row.partCode }}</td>
                <td class="pl-1">{{ row.itemName }}</td>
                <td class="text-center">{{ row.perUnit }}</td>
                <td class="text-center text-primary font-weight-bold">{{ row.totalNeed.toLocaleString() }}</td>
                <td class="text-center bg-gray-light">{{ row.stock.toLocaleString() }}</td>
                <td class="text-center bg-orange-light text-orange font-weight-bolder">{{ row.toProduce.toLocaleString() }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <div id="print-content" class="d-none">
      <div style="font-family: 'Sarabun', sans-serif; color: black; padding: 20px;">
        <h1 style="text-align: center; font-weight: bold; margin-bottom: 20px;">ใบสั่งผลิตชิ้นส่วน (Production Order)</h1>
        <div style="display: flex; justify-content: space-between; border-bottom: 2px solid #ccc; padding-bottom: 10px; margin-bottom: 15px;">
          <div style="font-size: 14px;">
            <p style="margin: 2px 0;"><strong>Project Order:</strong> {{ selectedPlan ? selectedPlan.po : '-' }}</p>
            <p style="margin: 2px 0;"><strong>SAP Code:</strong> {{ selectedPlan ? selectedPlan.sap : '-' }}</p>
            <p style="margin: 2px 0;"><strong>รายการ:</strong> {{ selectedPlan ? selectedPlan.name : '-' }}</p>
            <p style="margin: 2px 0;"><strong>สี (Color):</strong> {{ selectedPlan ? selectedPlan.color : '-' }}</p>
          </div>
          <div style="text-align: right; font-size: 14px;">
            <p style="margin: 2px 0; font-size: 1.4rem; font-weight: bold;">เป้าหมายวันนี้: {{ dailyTarget.toLocaleString() }} ตัว</p>
            <p style="margin: 2px 0;"><strong>วันที่สั่งผลิต:</strong> {{ new Date().toLocaleDateString('th-TH') }}</p>
          </div>
        </div>
        <table style="width: 100%; border-collapse: collapse; margin-bottom: 40px;">
          <thead>
            <tr>
              <th style="border: 1px solid black; padding: 8px; text-align: center; background-color: #f2f2f2;">#</th>
              <th style="border: 1px solid black; padding: 8px; text-align: center; background-color: #f2f2f2;">รหัสชิ้นส่วน</th>
              <th style="border: 1px solid black; padding: 8px; text-align: left; background-color: #f2f2f2; width: 35%;">รายการชิ้นส่วน</th>
              <th style="border: 1px solid black; padding: 8px; text-align: center; background-color: #f2f2f2;">ต่อตัว</th>
              <th style="border: 1px solid black; padding: 8px; text-align: center; background-color: #f2f2f2;">ต้องใช้</th>
              <th style="border: 1px solid black; padding: 8px; text-align: center; background-color: #f2f2f2;">สต็อก</th>
              <th style="border: 1px solid black; padding: 8px; text-align: center; background-color: #f2f2f2;">ผลิตเพิ่ม</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(row, index) in tableData" :key="index">
              <td style="border: 1px solid black; padding: 6px; text-align: center;">{{ index + 1 }}</td>
              <td style="border: 1px solid black; padding: 6px; text-align: center; font-weight: bold;">{{ row.partCode }}</td>
              <td style="border: 1px solid black; padding: 6px; text-align: left;">{{ row.itemName }}</td>
              <td style="border: 1px solid black; padding: 6px; text-align: center;">{{ row.perUnit }}</td>
              <td style="border: 1px solid black; padding: 6px; text-align: center; font-weight: bold;">{{ row.totalNeed.toLocaleString() }}</td>
              <td style="border: 1px solid black; padding: 6px; text-align: center;">{{ row.stock.toLocaleString() }}</td>
              <td style="border: 1px solid black; padding: 6px; text-align: center; font-weight: bold; font-size: 1.1rem;">{{ row.toProduce.toLocaleString() }}</td>
            </tr>
          </tbody>
        </table>
        <div style="display: flex; justify-content: space-between; text-align: center; margin-top: 50px;">
          <div style="width: 30%;">
            <p style="margin: 5px 0;">__________________________</p>
            <p style="margin: 5px 0;"><strong>ฝ่ายผลิต (ผู้รับงาน)</strong></p>
          </div>
        </div>
      </div>
    </div>
  </b-card>
</template>

<script>
import vSelect from 'vue-select'
import axios from 'axios'
import { BRow, BCol, BButton, BFormGroup, BCard, BFormInput, BPagination, BBadge } from 'bootstrap-vue'
import Ripple from 'vue-ripple-directive'

export default {
  components: { vSelect, BRow, BCol, BButton, BFormGroup, BCard, BFormInput, BPagination, BBadge },
  directives: { Ripple },
  data() {
    return {
      loading: false,
      allPlans: [],
      stockMap: {},
      bomDatabase: {},
      searchTemp: null,
      searchOptions: [],
      selectedPlan: null,
      dailyTarget: 0,
      tableData: [],
      filterPO: '',
      filterSAP: '',
      filterName: '',
      filterColor: '',
      currentPage: 1,
      perPage: 15
    }
  },
  computed: {
    filteredPlans() {
      return this.allPlans.filter(p => {
        return (p.po || '').toLowerCase().includes(this.filterPO.toLowerCase()) &&
               (p.sap || '').toLowerCase().includes(this.filterSAP.toLowerCase()) &&
               (p.name || '').toLowerCase().includes(this.filterName.toLowerCase()) &&
               (p.color || '').toLowerCase().includes(this.filterColor.toLowerCase());
      });
    },
    paginatedPlans() {
      const start = (this.currentPage - 1) * this.perPage;
      return this.filteredPlans.slice(start, start + this.perPage);
    }
  },
  mounted() {
    this.fetchData()
  },
  methods: {
    async fetchData(force = false) {
      this.loading = true
      try {
        const urlPlan = "https://docs.google.com/spreadsheets/d/e/2PACX-1vQUfe59fFqNK3a6yzya8ZB4W_2dqqBg0c_06ZpH3QQ65J53FfxoEO5iPFW9LEksMEexwLOx0XTb5UcF/pub?output=csv"
        const urlStock = "https://docs.google.com/spreadsheets/d/e/2PACX-1vShU_0KepJ0kBsIYP6zPhKVuZEE4TbCDyvx81yzPMBbdtO3SJp1kf1bRc4hbFfs4ErULD0oDyK39CkE/pub?output=csv"
        const urlBOM = "https://docs.google.com/spreadsheets/d/e/2PACX-1vT-sDov5Hl1nWrRBz9jCiiBeatitpYjZp0fqPYngNmvdY1teoDLQrhPBNkMZATc-rktKV37X2Q1qzkm/pub?output=csv"
        
        const [resPlan, resStock, resBOM] = await Promise.all([
          axios.get(urlPlan),
          axios.get(urlStock),
          axios.get(urlBOM)
        ])
        
        const tempStockMap = {}
        resStock.data.split(/\r?\n/).forEach((line, i) => {
          if (i === 0 || !line) return
          const cols = line.split(/,(?=(?:[^"]*"[^"]*")*[^"]*$)/)
          tempStockMap[(cols[1] || "").replace(/"/g, '').trim()] = parseInt((cols[7] || "0").replace(/[",]/g, '')) || 0
        })
        this.stockMap = tempStockMap

        const tempDB = {}
        let lastSap = ""
        resBOM.data.split(/\r?\n/).forEach((line, i) => {
          if (i === 0 || !line) return
          const c = line.split(/,(?=(?:[^"]*"[^"]*")*[^"]*$)/)
          let currentSap = (c[0] || "").replace(/"/g, '').trim();
          if (currentSap) lastSap = currentSap
          if (lastSap) {
            if (!tempDB[lastSap]) tempDB[lastSap] = []
            tempDB[lastSap].push({
              code: (c[2] || "").replace(/"/g, ''),
              name: (c[3] || "").replace(/"/g, ''),
              perUnit: parseFloat((c[5] || "0").replace(/[",]/g, '')) || 0
            })
          }
        })
        this.bomDatabase = tempDB

        this.allPlans = resPlan.data.split(/\r?\n/).slice(1).map(line => {
          if (!line) return null
          const c = line.split(/,(?=(?:[^"]*"[^"]*")*[^"]*$)/)
          const po = (c[2] || "").replace(/"/g, '')
          const sap = (c[3] || "").replace(/"/g, '')
          const qty = parseInt((c[8] || "0").replace(/[",]/g, '')) || 0
          
          const savedRemain = localStorage.getItem(`prod_remain_${po}_${sap}`)
          const remainingQty = savedRemain !== null ? parseInt(savedRemain) : qty

          return {
            po,
            sap,
            name: (c[6] || "").replace(/"/g, ''),
            color: (c[7] || "").replace(/"/g, ''),
            qty: qty,
            remainingQty: remainingQty,
            display: `PO: ${po} | SAP: ${sap} | ${c[6]}`
          }
        }).filter(v => v !== null).sort((a, b) => a.name.localeCompare(b.name, 'th'))
        
        this.searchOptions = this.allPlans.slice(0, 50)
      } catch (err) {
        console.error(err)
      } finally {
        this.loading = false
      }
    },
    
    onSearch(search, loading) {
      if (search.length) {
        const searchLower = search.toLowerCase()
        this.searchOptions = this.allPlans.filter(o => 
          o.display.toLowerCase().includes(searchLower)
        ).slice(0, 50)
      } else {
        this.searchOptions = this.allPlans.slice(0, 50)
      }
    },
    
    handleSelect(val) {
      if (val) {
        this.selectedPlan = val
        this.dailyTarget = val.remainingQty
        this.updateCalculation()
        this.$nextTick(() => { this.searchTemp = null })
      }
    },
    
    updateCalculation() {
      if (!this.selectedPlan) return
      const components = this.bomDatabase[this.selectedPlan.sap] || []
      this.tableData = components.map(item => {
        const need = item.perUnit * this.dailyTarget
        const stock = this.stockMap[item.code] || 0
        return {
          partCode: item.code,
          itemName: item.name,
          perUnit: item.perUnit,
          totalNeed: need,
          stock,
          toProduce: Math.max(0, need - stock)
        }
      })
    },
    
    clearSelection() {
      this.selectedPlan = null
      this.tableData = []
      this.dailyTarget = 0
    },

    clearLocalStorage() {
      if (confirm("⚠️ คุณต้องการล้างยอดที่เคยตัดทิ้งทั้งหมดในเครื่องนี้ใช่หรือไม่? ยอดจะกลับไปเท่ากับในตาราง Google Sheets")) {
        const keys = Object.keys(localStorage)
        keys.forEach(k => {
          if (k.startsWith('prod_remain_')) localStorage.removeItem(k)
        })
        this.fetchData()
        alert("ล้างข้อมูลสำเร็จ")
      }
    },
    
    printOrder() {
      if (!this.selectedPlan || this.dailyTarget <= 0) return;
      if (this.dailyTarget > this.selectedPlan.remainingQty) {
        alert("ยอดผลิตวันนี้เกินเป้าหมายคงเหลือ!");
        return;
      }

      const printWindow = window.open('', '_blank', 'width=800,height=600')
      const printContent = document.getElementById('print-content').innerHTML
      
      printWindow.document.write(`
        <!DOCTYPE html>
        <html>
        <head>
          <meta charset="UTF-8">
          <title>ใบสั่งผลิตชิ้นส่วน</title>
          <link href="https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;600;700&display=swap" rel="stylesheet">
          <style>
            body { font-family: 'Sarabun', sans-serif; margin: 0; padding: 20px; }
            @media print { body { margin: 0; } @page { margin: 1cm; } }
          </style>
        </head>
        <body>${printContent}</body>
        </html>
      `)
      
      printWindow.document.close()
      printWindow.focus()
      
      setTimeout(() => {
        printWindow.print()
        printWindow.close()

        const targetPlan = this.allPlans.find(p => p.po === this.selectedPlan.po && p.sap === this.selectedPlan.sap);
        if (targetPlan) {
          targetPlan.remainingQty -= this.dailyTarget;
          localStorage.setItem(`prod_remain_${targetPlan.po}_${targetPlan.sap}`, targetPlan.remainingQty);
          
          if (targetPlan.remainingQty <= 0) {
            alert("ผลิตครบตามเป้าหมายของ PO นี้แล้ว!");
            this.clearSelection();
          } else {
            this.dailyTarget = targetPlan.remainingQty;
            this.updateCalculation();
          }
        }
      }, 250)
    }
  }
}
</script>

<style lang="scss" scoped>
@import url('https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;600;700&display=swap');
@import '~vue-select/dist/vue-select.css';

::v-deep .custom-v-select {
  .vs__dropdown-toggle { border: 1px solid #d8d6de !important; border-radius: 0.357rem !important; padding: 6px 8px !important; background-color: #fff; min-height: 42px; }
  .vs__search::placeholder { color: #b9b9c3; font-size: 0.95rem; }
  .vs__search { margin-top: 0 !important; color: #6e6b7b; }
}

.filter-row-bg { background-color: #f3f2f7; }
.filter-input-style { border: 1px solid #d8d6de !important; border-radius: 4px !important; font-size: 0.8rem !important; }

.target-input-daily { border: 2px solid #ea5455 !important; font-size: 1.2rem; font-weight: 700; color: #ea5455; text-align: center; }
.target-card-fixed { background-color: #f8faff; border: 2px solid #7367f0; border-radius: 12px; padding: 10px; height: 100%; display: flex; flex-direction: column; justify-content: center; align-items: center; min-height: 85px; .target-label { font-size: 0.8rem; font-weight: 600; color: #5e5873; } .target-value { font-size: 1.6rem; font-weight: 800; } }
.cursor-pointer { cursor: pointer; }
.plan-row:hover { background-color: rgba(115, 103, 240, 0.08) !important; }

/* จัดการการตัดคำในตาราง */
.text-truncate {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.table-bom { 
  width: 100%; border-collapse: collapse; 
  th, td { border: 1px solid #dae1e7; padding: 10px 8px; vertical-align: middle; font-size: 0.85rem; } 
  th { background-color: #5d5fef; color: white; text-align: center; } 
  .bg-orange-light { background-color: rgba(255, 159, 67, 0.1); } 
  .text-orange { color: #ff9f43; font-weight: bold; } 
}

.d-none { display: none; }
</style>