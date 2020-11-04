<template>
  <div>
    <v-container>
      <v-row>
        <v-col>
          <v-card :style="statusSubmitRekap ? 'padding: 10px; border-color: #00cfb3; border-width: 2px;' : 'padding: 10px;'" outlined>
            <p><b>Rekap:</b></p>
            <v-file-input
              v-model="csvFile"
              :accept="sheetJSFT"
              outlined
              dense
              label="File input"
              @change="changeStatusSubmitRekap"
            />
            <div class="d-flex justify-center">
              <v-btn
                elevation="2"
                :disabled="!csvFile"
                @click="submitCsv"
              >
                Submit Rekap
              </v-btn>
            </div>
          </v-card>
        </v-col>
        <v-col>
          <v-card :style="statusSubmitData ? 'padding: 10px; border-color: #00cfb3; border-width: 2px;' : 'padding: 10px;'" outlined>
            <p><b>Data:</b></p>
            <v-file-input
              v-model="anotherCsv"
              outlined
              dense
              :accept="sheetJSFT"
              label="File input"
              @change="changeStatusSubmitData"
            />
            <div class="d-flex justify-center">
              <v-btn
                elevation="2"
                :disabled="!anotherCsv"
                @click="submitAnotherCsv"
              >
                Submit Data
              </v-btn>
            </div>
          </v-card>
        </v-col>
      </v-row>
      <br>
      <div class="d-flex justify-center">
        <v-btn
          color="primary"
          :disabled="Object.keys(fixData2).length === 0 || Object.keys(fixData).length === 0"
          @click="comparing"
        >
          Compare
        </v-btn>
      </div>
      <br>
      <v-row>
        <v-col>
          <v-card
            outlined
          >
            <v-container>
              <v-data-table
                :headers="headers"
                :items="result"
              >
                <template v-slot:top>
                  <v-toolbar flat>
                    <v-toolbar-title>Rekap yang tidak ada di Data</v-toolbar-title>
                  </v-toolbar>
                </template>
                <template v-slot:item.total="{ item }">
                  {{ item.total.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ".") }}
                </template>
              </v-data-table>
              <br>
              Total : Rp. {{ totals.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ".") }}
            </v-container>
          </v-card>
        </v-col>
        <v-col>
          <v-card outlined>
            <v-container>
              <v-data-table
                :headers="headers"
                :items="result2"
              >
                <template v-slot:top>
                  <v-toolbar flat>
                    <v-toolbar-title>Data yang tidak ada di Rekap</v-toolbar-title>
                  </v-toolbar>
                </template>
                <template v-slot:item.total="{ item }">
                  {{ item.total.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ".") }}
                </template>
              </v-data-table>
              <br>
              Total : Rp. {{ totals2.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ".") }}
            </v-container>
          </v-card>
        </v-col>
      </v-row>
      <br>
      <div>
        <v-card outlined>
          <v-container>
            <v-data-table
              :headers="headers2"
              :items="diffHarga"
            >
              <template v-slot:top>
                <v-toolbar flat>
                  <v-toolbar-title>Perbedaan total antara Rekap dan Data</v-toolbar-title>
                </v-toolbar>
              </template>
              <template v-slot:item.rekap="{ item }">
                {{ item.rekap.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ".") }}
              </template>
              <template v-slot:item.data="{ item }">
                {{ item.data.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ".") }}
              </template>
            </v-data-table>
          </v-container>
        </v-card>
      </div>
    </v-container>
    <v-dialog
      v-model="dialog"
      width="500"
    >
      <v-card>
        <v-card-title class="headline grey lighten-2">
          Penjelasan penggunaan aplikasi
        </v-card-title>

        <v-card-text>
          <br>
          - file hanya tabel saja, tanpa ada baris total / grand total
        </v-card-text>

        <v-divider />

        <v-card-actions>
          <v-spacer />
          <v-btn
            color="primary"
            text
            @click="dialog = false"
          >
            Mengerti
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>

<script>
import XLSX from 'xlsx'

export default {
  data: () => ({
    csvFile: null,
    anotherCsv: null,
    fixData: {},
    fixData2: {},
    statusSubmitRekap: false,
    statusSubmitData: false,
    result: [],
    result2: [],
    headers: [
      { text: 'NAMA', value: 'nama' },
      { text: 'REGBILL', value: 'regbill' },
      { text: 'TOTAL', value: 'total' }
    ],
    headers2: [
      { text: 'REGBILL', value: 'regbill' },
      { text: 'NAMA', value: 'nama' },
      { text: 'Total Rekap', value: 'rekap' },
      { text: 'Total Data', value: 'data' }
    ],
    sheetJSFT: ['xlsx', 'xlsb', 'xlsm', 'xls', 'xml', 'csv', 'txt', 'ods', 'fods', 'uos', 'sylk', 'dif', 'dbf', 'prn', 'qpw', '123', 'wb*', 'wq*', 'html', 'htm'].map((x) => { return '.' + x }).join(','),
    totals: 0,
    totals2: 0,
    diffHarga: [],
    dialog: true
  }),
  watch: {
    csvFile (data) {
      if (!data) {
        this.statusSubmitRekap = false
      }
    },
    anotherCsv (data) {
      if (!data) {
        this.statusSubmitData = false
      }
    }
  },
  methods: {
    changeStatusSubmitRekap () {
      this.statusSubmitRekap = false
    },
    changeStatusSubmitData () {
      this.statusSubmitData = false
    },
    comparing () {
      const status = {}
      const status2 = {}
      const diff = []
      const harga = []
      const harga2 = []
      this.totals = 0
      this.totals2 = 0

      for (const key in this.fixData) {
        if (this.fixData2[key]) {
          status[key] = true
          if (this.fixData[key].total !== this.fixData2[key].total) {
            diff.push({
              regbill: key,
              nama: this.fixData[key].nama,
              rekap: this.fixData[key].total,
              data: this.fixData2[key].total
            })
          }
        } else {
          status[key] = false
        }
      }

      for (const key in this.fixData2) {
        if (this.fixData[key]) {
          status2[key] = true
        } else {
          status2[key] = false
        }
      }

      for (const key in status) {
        if (!status[key]) {
          harga.push({
            nama: this.fixData[key].nama,
            regbill: key,
            total: this.fixData[key].total
          })
          this.totals = this.totals + this.fixData[key].total
        }
      }

      for (const key in status2) {
        if (!status2[key]) {
          harga2.push({
            nama: this.fixData2[key].nama,
            regbill: key,
            total: this.fixData2[key].total
          })
          this.totals2 = this.totals2 + this.fixData2[key].total
        }
      }

      this.result = [...harga]
      this.result2 = [...harga2]
    },
    submitAnotherCsv () {
      const objFix = {}
      const files = this.anotherCsv
      if (files) {
        const reader = new FileReader()
        reader.onload = (e) => {
          const bstr = e.target.result
          const wb = XLSX.read(bstr, { type: 'binary' })
          const wsname = wb.SheetNames[0]
          const ws = wb.Sheets[wsname]
          const data = XLSX.utils.sheet_to_json(ws, { header: 1 })
          const fixData = []
          let i = 0
          const header = {}
          while (data.length > i) {
            if (i === 0) {
              for (let j = 0; j < data[i].length; j++) {
                header[j] = data[i][j]
              }
            } else {
              const tempObj = {}
              for (let j = 0; j < Object.keys(header).length; j++) {
                tempObj[header[j]] = data[i][j]
              }
              fixData.push(tempObj)
            }
            i++
          }
          let j = 0
          while (j < fixData.length) {
            objFix[fixData[j].REGBILL] = {
              nama: fixData[j].NAMA,
              total: fixData[j]['Grand Total']
            }
            j++
          }
          this.fixData2 = { ...objFix }
          this.statusSubmitData = true
        }
        reader.readAsBinaryString(files)
      }
    },
    submitCsv () {
      const objectFix = {}
      const dataFix = []
      const files = this.csvFile
      if (files) {
        const reader = new FileReader()
        reader.onload = (e) => {
          const bstr = e.target.result
          const wb = XLSX.read(bstr, { type: 'binary' })
          const wsname = wb.SheetNames[0]
          const ws = wb.Sheets[wsname]
          const data = XLSX.utils.sheet_to_json(ws, { header: 1 })
          const fixData = []
          let i = 0
          const header = {}
          while (data.length > i) {
            if (i === 0) {
              for (let j = 0; j < data[i].length; j++) {
                header[j] = data[i][j]
              }
            } else {
              const tempObj = {}
              for (let j = 0; j < Object.keys(header).length; j++) {
                tempObj[header[j]] = data[i][j] || null
              }
              fixData.push(tempObj)
            }
            i++
          }

          let j = 0
          while (j < fixData.length) {
            if (fixData[j].NO && fixData[j].NO !== ' ') {
              dataFix.push({
                NO: parseInt(fixData[j].NO),
                NAMA: fixData[j].NAMA.trim(),
                TANGGAL: fixData[j].TANGGAL,
                'NO. BUKTI': fixData[j]['NO. BUKTI'],
                'NAMA TARIP': fixData[j]['NAMA TARIP'],
                JUMLAH: fixData[j].JUMLAH
              })
            } else if (fixData[j].JUMLAH || fixData[j]['NAMA TARIP']) {
              dataFix.push({
                NO: dataFix[dataFix.length - 1].NO,
                NAMA: dataFix[dataFix.length - 1].NAMA,
                TANGGAL: dataFix[dataFix.length - 1].TANGGAL,
                'NO. BUKTI': dataFix[dataFix.length - 1]['NO. BUKTI'],
                'NAMA TARIP': fixData[j]['NAMA TARIP'],
                JUMLAH: fixData[j].JUMLAH
              })
            }
            j++
          }
          let k = 0
          while (dataFix.length > k) {
            if (objectFix[dataFix[k]['NO. BUKTI']]) {
              objectFix[dataFix[k]['NO. BUKTI']].total = objectFix[dataFix[k]['NO. BUKTI']].total + dataFix[k].JUMLAH
            } else {
              objectFix[dataFix[k]['NO. BUKTI']] = {
                total: dataFix[k].JUMLAH
              }
            }
            objectFix[dataFix[k]['NO. BUKTI']].nama = dataFix[k].NAMA
            objectFix[dataFix[k]['NO. BUKTI']][dataFix[k]['NAMA TARIP']] = dataFix[k].JUMLAH
            k++
          }
          this.fixData = { ...objectFix }
          this.statusSubmitRekap = true
        }
        reader.readAsBinaryString(files)
      }
    }
  }
}
</script>
