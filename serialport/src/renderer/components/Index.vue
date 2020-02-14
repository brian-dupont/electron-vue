<template>
  <div>
    <div id="bottom">
      <div id="wrapper">
        <div id="left">
          <h1 v-if="this.isSerialPortOpen">{{ this.serialPort.path }} has been opened</h1>
          <h1 style="color:red;" v-else>No Serial Port Selected</h1>

          <serial-port-list @serial-port-selected="setSerialPort"></serial-port-list>

          <order-header :job-order="this.printRequest"></order-header>
        </div>

        <div id="right">
          <h6 style="font-size:24px;">Previous Weight:</h6>

          <div id="last-weight">{{ this.lastWeight }}</div>
        </div>
      </div>

      <div id="weight-list">
        <ul>
          <li v-for="(weight, index) in this.weightsReversed" :key="index" v-text="weight"></li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script>
import SerialPortList from "./SerialPortList";
import OrderHeader from "./OrderHeader";
const SerialPort = require("serialport");
const Readline = require("@serialport/parser-readline");
const fs = require("fs");
const express = require("express");
const expressApp = express();
const expressPort = 3000;
const open = require("open");

export default {
  name: "index",

  components: { SerialPortList, OrderHeader },

  data() {
    return {
      serialPort: null,
      isSerialPortOpen: false,
      printRequest: {},
      templateString: "",
      lastWeight: "",
      weights: []
    };
  },

  computed: {
    weightsReversed() {
      return this.weights.reverse();
    }
  },

  methods: {
    setSerialPort(path) {
      if (this.serialPort === null) {
        this.openPort(path);

        return;
      }

      if (this.serialPort.path === path) {
        return;
      }

      let self = this;

      if (this.serialPort.isOpen) {
        this.serialPort.close(() => (self.isSerialPortOpen = false));
      }

      this.openPort(path);
    },

    openPort(path) {
      this.serialPort = new SerialPort(path, {
        baudRate: 9600
      });

      let parser = this.serialPort.pipe(new Readline({ delimiter: "\n" }));
      let self = this;

      parser.on("data", this.handleIncomingSerialData);

      this.serialPort.on("error", function(error) {
        console.log(error.message);

        self.isSerialPortOpen = this.serialPort.isOpen ? true : false;
      });

      this.serialPort.on("open", () => {
        self.isSerialPortOpen = true;

        console.log(`${path} serial port has been opened.`);
      });
    },

    handleIncomingSerialData(data) {
      if (!this.printRequest.id) {
        alert(
          "No print request found in DB.  Did you press the 'label from scale' button in the ERP Dashboard?"
        );

        return false;
      }

      try {
        let weight = parseFloat(data);

        if (isNaN(weight)) {
          this.handleInvalidData(data);

          return;
        }

        this.handleValidData(weight);
      } catch (e) {
        console.log("Error handling the incoming serial data:");

        console.log(e.message);
      }
    },

    handleInvalidData(data) {
      alert(`Invalid Data: ${data}`);
    },

    handleValidData(weight) {
      this.lastWeight = weight;

      this.weights.push(this.lastWeight);

      this.$http
        .get(
          `http://192.168.1.12:8080/print_label_from_scale.php?print_request_id=${this.printRequest.id}&weight=${this.lastWeight}`
        )
        .then(res => {
          this.buildTemplateString(res.data);

          this.writeTemplateStringToFile(this.templateString);
        })
        .catch(err => console.log(err.message));
    },

    buildTemplateString(templateData) {
      this.printRequest.next_job_item_number =
        templateData.next_job_item_number;

      this.templateString = 
        `FORMATNAME=${this.printRequest.label_format}
        formatcount=${this.printRequest.lbl_count}
        where=(NUMBER= ${this.printRequest.item_number})
        SINGLEJOB=ON
        TESTPRINT=off
        NUMB="${templateData.next_job_item_number}"
        LOTN="${this.printRequest.lot_number}"
        WT=${templateData.weight}
        BAR=${templateData.barcode_without_next_job_item_number}
        ALLERGENS="${this.printRequest.allergens}"
        INPUT="${this.printRequest.input}"
        ;`;

        return this.templateString;
    },

    writeTemplateStringToFile(templateString, nextJobItemNumber) {
      fs.writeFile("serial.ser", nextJobItemNumber, err => {
        if (err) {
          console.log("Error writing to serial.ser file");

          return false;
        }

        fs.writeFile("label.CMD", templateString, err => {
          if (err) {
            console.log("Error writing to label.cmd file");

            return false;
          }
        });
      });
    },

    setPrintRequest(printRequest) {
      if (printRequest.type === "batch") {
        this.handleBatchRequest(printRequest.id);

        return;
      }

      this.printRequest = printRequest;
    },

    handleBatchRequest(printRequestId) {
      this.$http
        .get(
          `http://192.168.1.12:8080/print_label_from_batch.php?print_request_id=${printRequestId}`
        )
        .then(res => {
          this.printRequest = res.data;

          let templateString = this.buildTemplateString(this.printRequest, this.printRequest.next_job_item_number);

          this.writeTemplateStringToFile(templateString);
        })
        .catch(err => console.log(err));
    }
  },

  beforeCreate() {
    let bodyParser = require("body-parser");

    expressApp.use(bodyParser.json());
    expressApp.use(bodyParser.urlencoded({ extended: true }));

    expressApp.post("/setup-printing", (req, res) => {
      this.setPrintRequest(res.req.body);

      res.json({
        success: true
      });
    });

    expressApp.listen(expressPort, () => {});

    // Open EasyLabel
    open("C:\\Program Files (x86)\\Tharo\\EASYLABEL 5.12.2.1612\\easyshort")
      .then(res => console.log("EasyLabel opened"))
      .catch(err => console.log("Could not open EasyLabel", err));
  }
};
</script>

<style>
#wrapper {
  display: flex;
  justify-content: space-between;
}

#left,
#right {
  width: 50%;
  border: 1px solid black;
}

#right {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  font-size: 100px;
}

#weight-list {
  width: 100%;
  font-size: 30px;
}

h1 {
  font-size: 28px;
}
</style>