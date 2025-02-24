<!-- FileUploader.vue -->
<template>
  <div class="upload-container">
    <div
      class="upload-zone"
      @dragover.prevent="handleDragOver"
      @drop.prevent="handleDrop"
      :class="{ 'drag-over': isDragging }"
    >
      <input
        type="file"
        ref="fileInput"
        @change="handleFileSelect"
        multiple
        class="file-input"
      />
      <div class="upload-prompt">
        <p>Drop files here or click to select</p>
      </div>
    </div>

    <div class="file-list">
      <div v-for="file in files" :key="file.id" class="file-item">
        <div class="file-info">
          <span class="file-name">{{ file.name }}</span>
          <span class="file-size">({{ formatFileSize(file.size) }})</span>
        </div>

        <div class="progress-bar">
          <div
            class="progress"
            :style="{ width: file.progress + '%' }"
            :class="{ complete: file.progress === 100 }"
          ></div>
        </div>

        <div class="status">
          {{ file.status }}
          <button
            v-if="!file.complete"
            @click="cancelUpload(file)"
            class="cancel-btn"
          >
            Cancel
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from "axios";

export default {
  name: "FileUploader",

  props: {
    uploadUrl: {
      type: String,
      required: true,
    },
    maxFileSize: {
      type: Number,
      default: 50 * 1024 * 1024,
    },
    allowedTypes: {
      type: Array,
      default: () => [],
    },
  },

  data() {
    return {
      files: [],
      isDragging: false,
      uploadControllers: new Map(), // Store AbortController for each upload
    };
  },

  methods: {
    handleDragOver() {
      this.isDragging = true;
    },
    handleDrop(event) {
      this.isDragging = false;
      const droppedFiles = Array.from(event.dataTransfer.files);
      this.addFiles(droppedFiles);
    },
    handleFileSelect(event) {
      const selectedFiles = Array.from(event.target.files);
      this.addFiles(selectedFiles);
      this.$refs.fileInput.value = ""; // Reset input
    },

    validateFile(file) {
      if (file.size > this.maxFileSize) {
        return `File size exceeds ${this.formatFileSize(this.maxFileSize)}`;
      }

      if (this.allowedTypes.length > 0) {
        const isValidType = this.allowedTypes.some((type) => {
          if (type.startsWith(".")) {
            return file.name.toLowerCase().endsWith(type.toLowerCase());
          }
          return file.type.match(new RegExp(type.replace("*", ".*")));
        });

        if (!isValidType) {
          return "File type not allowed";
        }
      }

      return null;
    },

    addFiles(newFiles) {
      newFiles.forEach((file) => {
        const error = this.validateFile(file);

        const fileObj = {
          id: Date.now() + Math.random(),
          file,
          name: file.name,
          size: file.size,
          progress: 0,
          status: error || "Pending",
          complete: false,
          error: !!error,
        };

        this.files.push(fileObj);

        if (!error) {
          this.uploadFile(fileObj.id);
        }
      });
    },

    formatFileSize(bytes) {
      const sizes = ["Bytes", "KB", "MB", "GB"];
      if (bytes === 0) return "0 Bytes";
      const i = parseInt(Math.floor(Math.log(bytes) / Math.log(1024)));
      return Math.round(bytes / Math.pow(1024, i)) + " " + sizes[i];
    },

    async uploadFile(fileObjId) {
      const fileObj = this.files.find((file) => file.id === fileObjId);
      const controller = new AbortController();
      this.uploadControllers.set(fileObj.id, controller);

      try {
        fileObj.status = "Uploading...";

        const formData = new FormData();
        formData.append("file", fileObj.file);

        const response = await axios.post(this.uploadUrl, formData, {
          signal: controller.signal,
          onUploadProgress: (progressEvent) => {
            fileObj.progress = progressEvent.progress * 100;
          },
        });

        fileObj.progress = 100;
        fileObj.status = "Complete";
        fileObj.complete = true;

        this.uploadControllers.delete(fileObj.id);
        this.$emit("upload-complete", {
          file: fileObj,
          response: response.data,
        });
      } catch (error) {
        if (error.name === "AbortError") {
          fileObj.status = "Cancelled";
        } else {
          fileObj.status = `Error: ${error.message}`;
          fileObj.error = true;
          console.error("Upload error:", error);
          this.$emit("upload-error", { file: fileObj, error });
        }
      }
    },

    cancelUpload(fileObj) {
      const controller = this.uploadControllers.get(fileObj.id);
      if (controller) {
        controller.abort();
        this.uploadControllers.delete(fileObj.id);
      }
    },
  },
};
</script>

<style scoped>
.upload-container {
  max-width: 600px;
  margin: 20px auto;
}

.upload-zone {
  position: relative;
  padding: 40px;
  border: 2px dashed #ccc;
  border-radius: 8px;
  text-align: center;
  transition: border-color 0.3s ease;
}

.upload-zone.drag-over {
  border-color: #2196f3;
  background-color: rgba(33, 150, 243, 0.05);
}

.file-input {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  opacity: 0;
  cursor: pointer;
}

.upload-prompt {
  color: #666;
}

.file-list {
  margin-top: 20px;
}

.file-item {
  margin: 10px 0;
  padding: 15px;
  background: #f5f5f5;
  border-radius: 8px;
}

.file-info {
  margin-bottom: 10px;
}

.file-name {
  font-weight: bold;
}

.file-size {
  color: #666;
  margin-left: 8px;
}

.progress-bar {
  width: 100%;
  height: 8px;
  background: #e0e0e0;
  border-radius: 4px;
  overflow: hidden;
}

.progress {
  width: 0%;
  height: 100%;
  background: #4caf50;
  transition: width 0.3s ease;
}

.progress.complete {
  background: #2196f3;
}

.status {
  margin-top: 8px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  color: #666;
}

.cancel-btn {
  padding: 4px 8px;
  border: none;
  background: #ff5252;
  color: white;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
}

.cancel-btn:hover {
  background: #ff1744;
}
</style>
