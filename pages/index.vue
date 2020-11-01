<template>
  <div>
    <v-row>
      <v-col style="margin-left: 25px;" cols="1">
        Rekap:
      </v-col>
      <v-col>
        <v-file-input
          v-model="csvFile"
          accept="text/csv"
          label="File input"
        />
      </v-col>
      <v-col>
        <v-btn
          elevation="2"
          @click="submitCsv"
        >
          Submit
        </v-btn>
      </v-col>
    </v-row>
    <br>
    <v-row>
      <v-col style="margin-left: 25px;" cols="1">
        Data:
      </v-col>
      <v-col>
        <v-file-input
          v-model="anotherCsv"
          accept="text/csv"
          label="File input"
        />
      </v-col>
      <v-col>
        <v-btn
          elevation="2"
          @click="submitAnotherCsv"
        >
          Submit
        </v-btn>
      </v-col>
    </v-row>
    <br>
    <br>
    <v-btn
      :disabled="Object.keys(fixData2).length === 0 || Object.keys(fixData).length === 0"
      @click="comparing"
    >
      Compare
    </v-btn>
    <br>
    <v-row>
      <v-col>
        <b>Rekap yang tidak ada di Data</b>
        <v-data-table
          :headers="headers"
          :items="result"
          class="elevation-1"
        >
          <template v-slot:item.total="{ item }">
            {{ item.total.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ".") }}
          </template>
        </v-data-table>
        <br>
        Total : Rp. {{ totals.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ".") }}
      </v-col>
      <v-col>
        <b>Data yang tidak ada di Rekap</b>
        <v-data-table
          :headers="headers"
          :items="result2"
          class="elevation-1"
        >
          <template v-slot:item.total="{ item }">
            {{ item.total.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ".") }}
          </template>
        </v-data-table>
        <br>
        Total : Rp. {{ totals2.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ".") }}
      </v-col>
    </v-row>
    <br>
    <br>
    <b>Perbedaan total antara Rekap dan Data</b>
    <div>
      <v-data-table
        :headers="headers2"
        :items="diffHarga"
        class="elevation-1"
      >
        <template v-slot:item.rekap="{ item }">
          {{ item.rekap.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ".") }}
        </template>
        <template v-slot:item.data="{ item }">
          {{ item.data.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ".") }}
        </template>
      </v-data-table>
    </div>
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
          - file harus berextensi .csv
          <br>
          - format file .csv untuk rekap harus memiliki header NO, NAMA, TANGGAL, NO. BUKTI, NAMA TARIP, JUMLAH
          <br>
          - format file .csv untuk data harus memiliki header REGBILL, NAMA, Grand Total
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
import Papa from 'papaparse'

export default {
  data: () => ({
    csvFile: null,
    anotherCsv: null,
    fixData: {},
    fixData2: {},
    result: [],
    result2: [],
    headers: [
      { text: 'REGBILL', value: 'regbill' },
      { text: 'TOTAL', value: 'total' }
    ],
    headers2: [
      { text: 'REGBILL', value: 'regbill' },
      { text: 'Total Rekap', value: 'rekap' },
      { text: 'Total Data', value: 'data' }
    ],
    totals: 0,
    totals2: 0,
    diffHarga: [],
    dialog: true
  }),
  methods: {
    comparing () {
      const status = {}
      const status2 = {}
      const diff = []
      const harga = []
      const harga2 = []

      for (const key in this.fixData) {
        if (this.fixData2[key]) {
          status[key] = true
          if (this.fixData[key].total !== this.fixData2[key].total) {
            diff.push({
              regbill: key,
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
            regbill: key,
            total: this.fixData[key].total
          })
          this.totals = this.totals + this.fixData[key].total
        }
      }

      for (const key in status2) {
        if (!status2[key]) {
          harga2.push({
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
      const dataFix = []
      const objectFix = {}
      Papa.parse(this.anotherCsv, {
        header: true,
        transformHeader: (h) => {
          return h.trim()
        },
        step: (row) => {
          const { data } = row
          dataFix.push(data)
          const total = parseInt(data['Grand Total'].split(',').join(''))
          objectFix[data.REGBILL] = {
            total
          }
        },
        complete: () => {
          this.fixData2 = { ...objectFix }
        }
      })
    },
    submitCsv () {
      const dataFix = []
      const objectFix = {}
      Papa.parse(this.csvFile, {
        header: true,
        newline: '\\n',
        delimiter: ',',
        dynamicTyping: true,
        transformHeader: (h) => {
          return h.trim()
        },
        step: (row) => {
          const { data } = row
          if (data.NO && data.NO !== ' ') {
            dataFix.push({
              NO: parseInt(data.NO),
              NAMA: data.NAMA.trim(),
              TANGGAL: data.TANGGAL.trim(),
              'NO. BUKTI': data['NO. BUKTI'].trim(),
              'NAMA TARIP': data['NAMA TARIP'].trim(),
              JUMLAH: parseInt(data.JUMLAH.split(',').join(''))
            })
          } else {
            dataFix.push({
              NO: dataFix[dataFix.length - 1].NO,
              NAMA: dataFix[dataFix.length - 1].NAMA,
              TANGGAL: dataFix[dataFix.length - 1].TANGGAL,
              'NO. BUKTI': dataFix[dataFix.length - 1]['NO. BUKTI'],
              'NAMA TARIP': data['NAMA TARIP'].trim(),
              JUMLAH: parseInt(data.JUMLAH.split(',').join(''))
            })
          }
        },
        complete: (results) => {
          let i = 0
          while (dataFix.length > i) {
            if (objectFix[dataFix[i]['NO. BUKTI']]) {
              objectFix[dataFix[i]['NO. BUKTI']].total = objectFix[dataFix[i]['NO. BUKTI']].total + dataFix[i].JUMLAH
            } else {
              objectFix[dataFix[i]['NO. BUKTI']] = {
                total: dataFix[i].JUMLAH
              }
            }
            objectFix[dataFix[i]['NO. BUKTI']][dataFix[i]['NAMA TARIP']] = dataFix[i].JUMLAH
            i++
          }
          this.fixData = { ...objectFix }
        }
      })
    }
  }
}
</script>
