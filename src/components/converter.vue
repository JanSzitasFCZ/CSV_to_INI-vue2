<template>
  <div class="widget">
    <h1>EUROMAP63 CSV to INI Converter</h1>
    <div class="settings">
      <div class="setting">
        <label for="max-sessions">Max sessions:</label>
        <input id="max-sessions" v-model="maxSessions" />
      </div>
      <div class="setting">
        <label for="default-path">Default file path:</label>
        <input id="default-path" v-model="defaultPath" />
      </div>
      <div class="setting">
        <label for="default-path">Get .INI file:</label>
        <button @click="saveAsINI">Download</button>
      </div>
    </div>
    <div class="container">
      <textarea
        class="textbox big-input"
        id="input"
        ref="textarea1"
        v-model="leftText"
        style="height: 100px"
        @drop="handleFileDrop"
        placeholder="Drop file or paste text"
        @dragover.prevent
      ></textarea>
      <textarea
        class="textbox big-input"
        id="output"
        ref="textarea2"
        v-model="rightText"
        style="height: 100px; resize: none"
        placeholder="INI will be generated here"
        readonly
      ></textarea>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      maxSessions: "15",
      defaultPath: "C:\\FANUC\\EM63\\SESSION\\",
      leftText: "",
      rightText: "",
      textarea1: null,
    };
  },
  // Setting the result textbox to resize according to the input textbox
  mounted() {
    // Initialize ResizeObserver
    const input = this.$refs.textarea1;
    const output = document.getElementById("output");
    const resizeObserver = new ResizeObserver(() => {
      const computedStyles = getComputedStyle(input);
      const paddingTop = parseInt(
        computedStyles.getPropertyValue("padding-top"),
        10
      );
      const paddingBottom = parseInt(
        computedStyles.getPropertyValue("padding-bottom"),
        10
      );
      const borderTop = parseInt(
        computedStyles.getPropertyValue("border-top-width"),
        10
      );
      const borderBottom = parseInt(
        computedStyles.getPropertyValue("border-bottom-width"),
        10
      );

      // Calculate the actual content height
      const actualHeight =
        input.offsetHeight -
        paddingTop -
        paddingBottom -
        borderTop -
        borderBottom;

      output.style.height = actualHeight + "px";
    });
    resizeObserver.observe(input);
  },

  watch: {
    // Watch for changes in the leftText property
    leftText: {
      handler: "runConversion", // Call the runConversion method when leftText changes
      immediate: false, // Trigger the handler immediately on component creation
    },
  },

  methods: {
    // Executes conversion from the left textbox to the right textbox and implements checks
    runConversion() {
      const isValidMaxSessions = this.isInteger(this.maxSessions);
      const csvData = this.leftText;
      const delimiter = ",";
      // Check if max sessions is a number
      if (!isValidMaxSessions) {
        alert("Note that Max Sessions setting is not a number!");
        return;
      }
      // Check if input is valid CSV
      if (this.validateCsv(csvData, delimiter)) {
      } else {
        alert("Invalid CSV structure");
        this.leftText = "";
        return;
      }
      // If checks passed, execute the conversion
      this.rightText = this.csvToCustomFormat(this.leftText, ",");
    },

    // Creates an output .ini file which is downloaded by simulating a click to a link
    saveAsINI() {
      const blob = new Blob([this.rightText], { type: "text/plain" });
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = "output.ini";
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      URL.revokeObjectURL(url);
    },

    // Check validity of input data by counting columns in the header and rows
    validateCsv(data, delimiter) {
      try {
        // Split the data into lines
        const lines = data.split("\n");

        // Get the header line to calculate the number of columns
        const headerLine = lines[0].trim();
        const headerFields = headerLine.split(delimiter);
        const numFields = headerFields.length;

        // Validate the remaining rows by counting the number of columns and comparing to the header
        for (let i = 1; i < lines.length; i++) {
          const row = lines[i].trim();
          if (row) {
            const rowFields = row.split(delimiter);
            if (rowFields.length !== numFields) {
              return false;
            }
          }
        }

        return true;
      } catch (error) {
        return false;
      }
    },

    // Function to validate if input is an integer
    isInteger(value) {
      try {
        const intValue = parseInt(value, 10);
        if (isNaN(intValue)) {
          return false;
        }
        return true;
      } catch (error) {
        return false;
      }
    },

    // Processing of text insertion if an input file is drag-and-dropped
    handleFileDrop(event) {
      // Prevent opening the file in a browser itself
      event.preventDefault();
      const file = event.dataTransfer.files[0];

      if (file) {
        const reader = new FileReader();
        reader.onload = () => {
          const csvData = reader.result;
          const delimiter = ",";

          if (this.validateCsv(csvData, delimiter)) {
            this.leftText = csvData;
          } else {
            // Display a browser dialog (message) for invalid CSV structure
            alert("Invalid CSV structure");
          }
        };
        reader.readAsText(file);
      }
    },

    // Converts CSV to INI
    csvToCustomFormat(inputData, delimiter) {
      // Removes quotation marks from the file
      inputData = inputData.replace(/['"]+/g, "");

      const machineData = [];
      const machineInfoMap = {};

      const lines = inputData.trim().split("\n");
      const headers = lines[0].split(delimiter);

      const machineIdIndex = headers.indexOf("Machine ID");
      const ipAddressIndex = headers.indexOf("IP address");
      const mngIndex = headers.indexOf("Mng.");

      for (let i = 1; i < lines.length; i++) {
        const values = lines[i].split(delimiter);
        const mngValue = values[mngIndex];

        // Check if the "Mng." column value is "Out of Mng." if yes, exclude this line from the output
        if (mngValue !== "Out of Mng.") {
          const machineId = values[machineIdIndex];
          const paddedMachineId = machineId.padStart(2, "0");
          const machineName = `M${paddedMachineId}`;
          machineData.push(machineName);

          machineInfoMap[machineName] = {
            IPADDRESS: values[ipAddressIndex],
            MAXSESSIONS: this.maxSessions,
            SESSIONPATH: this.defaultPath + `M${paddedMachineId}`,
          };
        }
      }

      let customContent = "[MACHINES]\n";

      for (let i = 0; i < machineData.length; i++) {
        customContent += `${i + 1}=${machineData[i]}\n`;
      }
      customContent += "\n";

      for (const machineName of machineData) {
        const machineInfo = machineInfoMap[machineName];
        customContent += `[${machineName}]\n`;

        for (const [key, value] of Object.entries(machineInfo)) {
          customContent += `${key}=${value}\n`;
        }
        customContent += "\n";
      }

      return customContent;
    },
  },
};
</script>

<style scoped>
/* CSS styles for your component here */
body {
  margin: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  font-family: Arial, sans-serif;
}

.container {
  display: flex;
  justify-content: left;
  align-items: center;
  width: 100%;
  height: 100%;
  margin-top: 20px;
}

.big-input {
  width: 50%;
  padding: 10px;
  margin-right: 1%;
}

.settings {
  margin-bottom: 20px;
  display: flex;
  justify-content: left;
}

.setting {
  display: flex;
  flex-direction: column;
  width: 30%;
  margin-right: 5%;
}

.setting label {
  margin-bottom: 3px;
}

.setting input {
  width: 100%;
  padding: 5px;
  box-sizing: border-box;
}

.widget {
  width: 100%;
  height: 100%;
}
</style>
