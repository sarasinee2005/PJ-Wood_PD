<template>
  <div class="production-plan">
    <b-row>
      <b-col cols="12">
        <header class="text-center mb-4">
          <h1 class="display-4 font-weight-bold text-dark">รายงานการผลิตและสถานะ </h1>
          <div class="last-update-badge">
            {{ lastUpdateText }}
          </div>
        </header>

        <div class="po-section mb-4">
          <div class="po-header d-flex justify-content-between align-items-center mb-3">
            <h3 class="text-danger m-0">🚩 รายการที่ติด PO# / PP#</h3>
            <b-button 
              variant="danger" 
              pill 
              size="sm" 
              @click="isPoExpanded = !isPoExpanded"
            >
              {{ isPoExpanded ? '🔼 ย่อรายการ' : '🔽 ขยายรายการทั้งหมด' }}
            </b-button>
          </div>
          
          <div :class="['po-dashboard', { 'collapsed': !isPoExpanded }]">
            <div v-if="poItems.length === 0" class="text-muted p-3">ไม่มีรายการติด PO ในระบบ</div>
            <div v-for="(item, index) in poItems" :key="index" class="po-card">
              <small>📅 {{ item[0] }}</small>
              <b class="d-block my-1">{{ item[11] }}</b>
              <small>{{ item[1] }} | {{ item[2] }}</small>
            </div>
          </div>
        </div>

        <div class="filter-section d-flex align-items-center justify-content-center gap-3 mb-4">
          <label class="font-weight-bold m-0 text-dark">🔎 ค้นหา (วันที่ผลิต):</label>
          <flat-pickr
            v-model="filterDate"
            :config="fpConfig"
            class="form-control custom-date-input"
            placeholder="เลือกวันที่"
          />
          <b-button variant="primary" @click="resetFilter">แสดงทั้งหมด</b-button>
          <div class="ml-3 text-dark">
            พบ: <span class="row-count">{{ filteredData.length }}</span> รายการ
          </div>
        </div>

        <div class="table-container shadow-lg">
          <table class="custom-table">
            <thead>
              <tr>
                <th>วันที่ผลิต</th>
                <th>ลูกค้า</th>
                <th>รายการ</th>
                <th style="width: 200px;">รายละเอียด</th>
                <th>PO</th>
                <th>ข้อมูล</th>
                <th>จำนวน</th>
                <th>จำนวนผลิต</th>
                <th>หน่วย</th>
                <th style="color: #f6ad55;">Output Price</th>
                <th>หมายเหตุ</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(row, idx) in filteredData" :key="idx">
                <td>{{ row[0] || '-' }}</td>
                <td>{{ row[1] || '-' }}</td>
                <td>{{ row[2] || '-' }}</td>
                <td class="text-description">{{ row[3] || '-' }}</td>
                <td>{{ row[4] || '-' }}</td>
                <td>{{ row[6] || '-' }}</td>
                <td class="num-cell">{{ row[8] || '-' }}</td>
                <td class="num-cell">{{ row[9] || '-' }}</td>
                <td>{{ row[10] || '-' }}</td>
                <td class="price-cell">{{ formatPrice(row[20]) }}</td>
                <td>{{ row[11] || '-' }}</td>
              </tr>
              <tr v-if="filteredData.length === 0">
                <td colspan="11" class="text-center p-5 text-danger">
                  ❌ ไม่พบข้อมูลสำหรับวันที่เลือก
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        </b-col>
    </b-row>
  </div>
</template>

<script>
import { BRow, BCol, BButton } from 'bootstrap-vue'
import flatPickr from 'vue-flatpickr-component'
import 'flatpickr/dist/flatpickr.css'

export default {
  components: {
    BRow,
    BCol,
    BButton,
    flatPickr,
  },
  data() {
    return {
      CSV_URL: "https://docs.google.com/spreadsheets/d/e/2PACX-1vQUfe59fFqNK3a6yzya8ZB4W_2dqqBg0c_06ZpH3QQ65J53FfxoEO5iPFW9LEksMEexwLOx0XTb5UcF/pub?output=csv",
      masterData: [],
      filterDate: null,
      lastUpdateText: 'กำลังโหลดข้อมูล...',
      isPoExpanded: false,
      autoRefreshInterval: null,
      fpConfig: {
        dateFormat: "m/d/Y",
        allowInput: true,
      }
    }
  },
  computed: {
    filteredData() {
      if (!this.filterDate) return this.masterData;
      const targetVal = this.getDateValue(this.filterDate);
      return this.masterData.filter(row => this.getDateValue(row[0]) === targetVal);
    },
    poItems() {
      return this.masterData
        .filter(r => (r[11] || "").includes("ติด PO") || (r[11] || "").includes("ติด PP"))
        .reverse();
    }
  },
  methods: {
    async fetchData() {
      try {
        const response = await fetch(`${this.CSV_URL}&t=${Date.now()}`);
        const text = await response.text();
        const lines = text.split(/\r?\n/).filter(l => l.trim() !== "");
        
        this.masterData = lines.slice(1).map(line => {
          return line.split(/,(?=(?:(?:[^"]*"){2})*[^"]*$)/).map(cell => 
            cell.replace(/^"|"$/g, '').trim()
          );
        });
        
        this.lastUpdateText = "อัปเดตล่าสุด: " + new Date().toLocaleTimeString('th-TH');
      } catch (e) {
        console.error("Fetch error:", e);
        this.lastUpdateText = "โหลดข้อมูลไม่สำเร็จ";
      }
    },
    getDateValue(str) {
      if (!str || str === "-" || str === "") return null;
      let cleanStr = str.trim().replace(/-/g, '/');
      const parts = cleanStr.split('/');
      if (parts.length !== 3) return null;

      let d = parseInt(parts[0]);
      let m = parseInt(parts[1]);
      let y = parseInt(parts[2]);

      if (y > 2500) y -= 543;
      if (y < 100) y += 2000;
      return (y * 10000) + (m * 100) + d;
    },
    formatPrice(val) {
      if (!val || val === "-") return "-";
      const num = Number(val.toString().replace(/,/g, ''));
      return isNaN(num) ? val : num.toLocaleString();
    },
    resetFilter() {
      this.filterDate = null;
    }
  },
  mounted() {
    this.fetchData();
    this.autoRefreshInterval = setInterval(this.fetchData, 60000);
  },
  beforeDestroy() {
    if (this.autoRefreshInterval) clearInterval(this.autoRefreshInterval);
  }
}
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;600;700&display=swap');

* { box-sizing: border-box; font-family: 'Sarabun', sans-serif; }

/* 1. แก้ช่องวันที่ให้สั้นลง */
.custom-date-input {
  width: 180px !important;
  flex: none !important;
  text-align: center;
  font-weight: bold;
}

/* 2. จัดการตารางให้กึ่งกลางเป๊ะทั้งแนวตั้งและแนวนอน */
.custom-table {
  width: 100%;
  border-collapse: collapse;
  min-width: 1300px;
}

.custom-table th {
  background-color: #f1f5f9;
  color: #2d3748;
  padding: 15px 10px;
  font-size: 1rem;
  border-bottom: 2px solid #2d3748;
  text-align: center !important;   /* กึ่งกลางแนวนอน */
  vertical-align: middle !important; /* กึ่งกลางแนวตั้ง */
  position: sticky;
  top: 0;
  z-index: 10;
}

.custom-table td {
  padding: 14px 10px;
  border-bottom: 1px solid #e2e8f0;
  color: #2d3748;
  font-size: 0.95rem;
  text-align: center !important;    /* กึ่งกลางแนวนอน */
  vertical-align: middle !important;  /* กึ่งกลางแนวตั้ง */
}

/* ปรับแต่งคอลัมน์พิเศษ */
.text-description {
  max-width: 250px;
  word-break: break-word; /* ถ้าข้อความยาวให้ตัดบรรทัดแต่ยังอยู่กึ่งกลาง */
}

.num-cell { font-weight: bold; color: #3182ce; font-size: 1.05rem; } 
.price-cell { font-weight: bold; color: #dd6b20; font-size: 1.05rem; }

/* ส่วนอื่นๆ คงเดิม */
header h1 { font-size: 2.2rem; font-weight: 700; margin-bottom: 5px; }
.last-update-badge {
    font-size: 0.9rem; color: #fff; background: #4a5568;
    padding: 5px 20px; border-radius: 50px; display: inline-block;
}
.filter-section { 
    background: #f7fafc; padding: 20px; border-radius: 15px; 
    border: 1px solid #2d3748; margin: 20px 0;
}
.po-header { border-bottom: 2px solid #e74c3c; padding-bottom: 8px; }
.po-dashboard { display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 15px; overflow: hidden; transition: max-height 0.4s ease; }
.po-dashboard.collapsed { max-height: 150px; }
.po-card { background: #fff; border-left: 5px solid #e74c3c; padding: 15px; border-radius: 10px; border: 1px solid #2d3748; }
.table-container { border-radius: 12px; overflow: auto; border: 1px solid #2d3748; background: #fff; }
.row-count { font-weight: bold; color: #38b2ac; font-size: 1.2rem; }
</style>