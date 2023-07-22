<template>
  <div class="center feed q-mt-xl">
    <div class="">
      <q-input
        v-model="proceedingNumber"
        label="Enter the proceeding number"
        outlined
      >
        <!-- at the end of input, insert button -->
        <template v-slot:append>
          <q-btn color="primary" label="Download" @click="downloadDocuments" />
        </template>
      </q-input>
    </div>
    <!-- Add this line to display the summary and PDF -->
    <q-card v-if="summary" class="q-mt-lg">
      <q-card-section> Summary: {{ summary }} </q-card-section>
      <q-separator inset />
      <q-card-section>
        <!-- Display the PDF here -->
        <iframe :src="pdfSrc" style="width: 100%; height: 1080px;"></iframe>
      </q-card-section>
    </q-card>
  </div>
</template>

<script>
import { saveAs } from "file-saver";
import { getDocument, GlobalWorkerOptions } from "pdfjs-dist";
import axios from "axios";
import JSZip from "jszip";

GlobalWorkerOptions.workerSrc =
  "../node_modules/pdfjs-dist/build/pdf.worker.js";

export default {
  data() {
    return {
      proceedingNumber: null,

      OPENAI_API_KEY: "",

      summary: "",
      originalText: "", // Add this line to store the original text
      pdfSrc: null,
    };
  },
  methods: {
    async downloadDocuments() {
      const api_url = `https://developer.uspto.gov/ptab-api/documents.zip/${this.proceedingNumber}/download`;

      try {
        const response = await axios({
          method: "GET",
          url: api_url,
          responseType: "arraybuffer",
        });

        // Extract PDF from ZIP file
        const zip = new JSZip();
        const zipData = await zip.loadAsync(response.data);
        const pdfFileName = Object.keys(zipData.files)[0]; // Assumes the PDF is the first file in the ZIP
        const pdfData = await zipData.files[pdfFileName].async("arraybuffer");

        this.pdfSrc = URL.createObjectURL(new Blob([pdfData], { type: 'application/pdf' }));

        // Save the PDF file
        // saveAs(new Blob([pdfData]), "document.pdf");
        this.$q.notify("PDF file downloaded successfully.");

        // Convert downloaded PDF to tokenized text
        const tokenizedText = await this.convertPdfToTokenizedText(pdfData);
        if (tokenizedText) {
          // Pass the text to the ChatGPT API to generate a summary
          this.completionCall(tokenizedText);
        } else {
          this.$q.notify({
            color: "negative",
            message: "Failed to convert PDF to tokenized text.",
          });
        }
      } catch (error) {
        this.$q.notify({
          color: "negative",
          message: `Download failed: ${error}`,
        });
      }
    },

    blobToArrayBuffer(blob) {
      return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.onloadend = () => resolve(reader.result);
        reader.onerror = reject;
        reader.readAsArrayBuffer(blob);
      });
    },

    async convertPdfToTokenizedText(file) {
      try {
        const loadingTask = getDocument(file);
        const pdf = await loadingTask.promise;
        const totalPages = pdf.numPages;

        let tokenizedText = "";

        for (let pageNumber = 1; pageNumber <= totalPages; pageNumber++) {
          const page = await pdf.getPage(pageNumber);
          const content = await page.getTextContent();
          const text = content.items.map((item) => item.str).join(" ");
          tokenizedText += `${text}\n`;
        }

        this.originalText = tokenizedText; // Update originalText here

        return tokenizedText;
      } catch (error) {
        console.error("Error converting PDF to tokenized text:", error);
        return null;
      }
    },

    saveTextFile(text) {
      const blob = new Blob([text], { type: "text/plain;charset=utf-8" });
      saveAs(blob, "tokenized_text.txt");
    },

    async completionCall(input) {
      // console.log("input", input);
      let messages = [];
      messages.push({ role: "user", content: `Summarize: ${input}` });

      const apiKey = process.env.OPENAI_API_KEY || this.OPENAI_API_KEY;
      const apiEndpoint = "https://api.openai.com/v1/chat/completions"; // Update the endpoint according to the API version you're using

      const headers = {
        "Content-Type": "application/json",
        Authorization: `Bearer ${apiKey}`,
      };

      const data = {
        model: "gpt-3.5-turbo-16k", // USE THIS MODEL FOR NOW FOR FASTER RESPONSES
        messages: messages,
      };

      let attempts = 3;
      let success = false;

      while (attempts > 0 && !success) {
        try {
          const response = await axios.post(apiEndpoint, data, { headers });
          const completion = response.data;
          this.summary = completion.choices[0].message.content;
          messages.push({
            role: completion.choices[0].message.role,
            content: this.summary,
          });
          // console.log(this.summary);
          success = true;
        } catch (error) {
          console.error("API call failed:", error);
          attempts--;
          if (attempts > 0) {
            console.log("Retrying... attempts left:", attempts);
          }
        }
      }

      if (!success) {
        console.error("Failed after 3 attempts");
        // Handle the case when all attempts fail
      }
    },
  },
};
</script>
<style scoped>
.center {
  margin-left: auto;
  margin-right: auto;
}
.feed {
  max-width: 1000px;
}
</style>
