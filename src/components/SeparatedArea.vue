<template>
  <div id="drop-area">
    <div class="control-area">
      <input
        type="file"
        name="fields[assetsFieldHandle][]"
        id="assetsFieldHandle"
        @change="onChange"
        ref="file"
        accept=".xls, .xlsx, .csv"
      />
      <v-btn small color="primary" @click="callApi">call api</v-btn>
    </div>
    <div
      @dragover="dragOver"
      @dragleave="dragLeave"
      @drop="drop"
      class="drop-rect"
    ></div>
    <div class="drop-area">
      <v-data-table :headers="headers" :items="json_array" class="elevation-1">
        <template slot="headerCell" slot-scope="props">
          <v-tooltip bottom>
            <span slot="activator">
              {{ props.header.text }}
            </span>
            <span>
              {{ props.header.text }}
            </span>
          </v-tooltip>
        </template>
        <template slot="items" slot-scope="props">
          <td class="text-xs-right">{{ props.item.name }}</td>
          <td class="text-xs-right">{{ props.item.address }}</td>
          <td class="text-xs-right">{{ props.item.city }}</td>
          <td class="text-xs-right">{{ props.item.state }}</td>
          <td class="text-xs-right">{{ props.item.zip }}</td>
          <td class="text-xs-right">{{ props.item.lat }}</td>
          <td class="text-xs-right">{{ props.item.lng }}</td>
        </template>
      </v-data-table>
    </div>
    <div class="common csv-area">
      <textarea
        @paste="onPaste"
        @blur="onBlur"
        v-model="csv_data"
        ref="csvarea"
      ></textarea>
    </div>
  </div>
</template>

<script>
import XLSX from "xlsx";
import csv2json from "csvjson-csv2json";
import { Parser } from "json2csv";
import axios from "axios";

export default {
  name: "DropArea",
  delimiters: ["${", "}"],
  data: function() {
    return {
      file: {},
      json_array: [],
      headers: [
        {
          align: "center",
          sortable: false,
          text: "Name",
          value: "name",
          width: "5%",
        },
        {
          align: "center",
          sortable: false,
          text: "Address",
          value: "address",
          width: "25%",
        },
        {
          align: "center",
          sortable: false,
          text: "City",
          value: "city",
          width: "10%",
        },
        {
          align: "center",
          sortable: false,
          text: "State",
          value: "state",
          width: "5%",
        },
        {
          align: "center",
          sortable: false,
          text: "Zip",
          value: "zip",
          width: "5%",
        },
        {
          align: "center",
          sortable: false,
          text: "Lat",
          value: "lat",
          width: "10%",
        },
        {
          align: "center",
          sortable: false,
          text: "Lng",
          value: "lng",
          width: "10%",
        },
      ],
      csv_data: "",
    };
  },
  mounted: function() {
    this.hideCSVArea();
    const json2csvParser = new Parser();
    this.csv_data = json2csvParser.parse(this.json_array);
  },
  methods: {
    getSheetHeader(sheet) {
      let headers = [];
      let range = XLSX.utils.decode_range(sheet["!ref"]);
      let C,
        R = range.s.r; /* start in the first row */
      /* walk every column in the range */
      for (C = range.s.c; C <= range.e.c; ++C) {
        let cell =
          sheet[
            XLSX.utils.encode_cell({ c: C, r: R })
          ]; /* find the cell in the first row */

        let hdr = "UNKNOWN " + C; // <-- replace with your desired default
        if (cell && cell.t) hdr = XLSX.utils.format_cell(cell);

        headers.push(hdr);
      }

      this.headers = [...headers, "Lat", "Lng"];
    },
    onChange() {
      this.file = this.$refs.file.files[0];
      let self = this;
      let reader = new FileReader();
      reader.onload = async function(e) {
        let data = new Uint8Array(e.target.result);
        let workbook = XLSX.read(data, { type: "array" });
        let sheetName = workbook.SheetNames[0];
        let worksheet = workbook.Sheets[sheetName];
        let tmp = await XLSX.utils.sheet_to_json(worksheet);
        self.json_array = await self.convertKeysToLowerCase(tmp);
        self.csv_data = await XLSX.utils.sheet_to_csv(worksheet);
        // self.getSheetHeader(worksheet);
      };
      reader.readAsArrayBuffer(this.file);
    },
    convertKeysToLowerCase(arr) {
      let output = [];
      for (let i = 0; i < arr.length; i++) {
        let tmp = {};
        for (let obj of Object.entries(arr[i])) {
          if (!obj) continue;
          let key = obj[0].toString().toLowerCase();
          let val = obj[1].toString();
          tmp[key] = val;
        }
        output.push(tmp);
      }

      console.log(output, "===output");

      return output;
    },
    async callApi() {
      let json_array_tmp = [...this.json_array];
      // this.headers = [...this.headers, "Lat", "Lng"];
      // this.headers.splice(0, this.headers.length, ...new Set(this.headers));

      for (let i = 0; i < json_array_tmp.length; i++) {
        let item = this.json_array[i];
        let city = "",
          state = "",
          postalcode = "";
        for (let obj of Object.entries(item)) {
          if (!obj) continue;

          if (obj[0].toString().toLowerCase() === "city")
            city = `&city=${obj[1]}`;
          if (obj[0].toString().toLowerCase() === "state")
            state = `&state=${obj[1]}`;
          if (obj[0].toString().toLowerCase() === "zip")
            postalcode = `&postalcode=${obj[1]}`;
        }

        let count = 0;
        let maxTries = 100;
        // eslint-disable-next-line no-constant-condition
        while (true) {
          try {
            let url =
              "https://us1.locationiq.com/v1/search.php?key=pk.5583d733f08dd889b77df42f1d00337a&format=json&" +
              city +
              state +
              postalcode;

            let loc = await axios.get(url);

            item.lat = loc.data[0].lat;
            item.lng = loc.data[0].lon;
            // console.log(url);
            break;
          } catch (e) {
            if (++count == maxTries)
              console.log(e, maxTries, "times tried, but failed!");
          }
        }

        this.json_array[i] = item;
        this.json_array = [...this.json_array];
      }
    },
    dragOver(event) {
      event.preventDefault();
      this.showEffect();
    },
    dragLeave(event) {
      event.preventDefault();
      this.resetEffect();
    },
    drop(event) {
      event.preventDefault();
      this.$refs.file.files = event.dataTransfer.files;
      this.resetEffect();
      this.onChange();
    },
    showEffect() {
      document.querySelector(".drop-rect").style.border = "3px green dashed";
    },
    resetEffect() {
      document.querySelector(".drop-rect").style.border = "1px gray dashed";
    },
    onPaste(event) {
      this.csv_data = event.target.value;
    },
    async onBlur(event) {
      this.hideCSVArea();
      try {
        this.json_array = await csv2json(event.target.value, {
          parseNumbers: true,
        });
      } catch (e) {
        console.log(e);
      }
    },
    rectClick() {
      // this.showCSVArea();
      // this.$refs.csvarea.focus();
      // this.$refs.csvarea.select();
    },
    showCSVArea() {
      document.querySelector(".csv-area").style.display = "";
    },
    hideCSVArea() {
      document.querySelector(".csv-area").style.display = "none";
    },
  },
};
</script>
<style lang="scss">
[v-cloak] {
  display: none;
}
#drop-area {
  display: flex;
  flex-direction: column;
  .drop-area,
  .drop-rect,
  .control-area {
    margin: auto;
    top: 0;
    right: 0;
    bottom: 0;
    left: 0;
  }
  .control-area {
    padding-bottom: 30px;
    padding-top: 50px;
  }
  .drop-rect {
    border: gray 1px dashed;
    width: 50%;
    height: 100px;
    &:hover {
      border: 3px green dashed;
    }
  }
  .drop-area {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    width: 90%;
    height: 75%;
    padding-top: 30px;
    .common {
      width: 100%;
      height: 100%;
    }
    .table-div {
      overflow: auto;
      background: white;
      border-radius: 5px;
      border: 1px gray solid;
      display: flex;
      justify-content: center;
      td {
        border-bottom: 0;
        margin: 0;
        padding: 0;
        input {
          padding: 5;
          margin: 0;
        }
      }
      th.active {
        text-align: center;
        border: 1px gray solid;
      }
      th.error {
        text-align: center;
        border: 0; //1px gray solid;
      }
    }
    .csv-area {
      textarea {
        width: 100%;
        height: 100%;
        &:focus {
          background: white;
        }
      }
    }
  }
}
</style>