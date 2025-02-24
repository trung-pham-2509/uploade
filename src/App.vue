<template>
  <div>
    <h1>File Upload Demo</h1>
    <FileUploader
      upload-url="https://api.escuelajs.co/api/v1/files/upload"
      :max-file-size="6 * 1024 * 1024"
      :allowed-types="['image/*', '.pdf']"
      @upload-complete="handleUploadComplete"
      @upload-error="handleUploadError"
    />
  </div>
</template>

<script>
import FileUploader from "./components/FileUploader.vue";

export default {
  components: {
    FileUploader,
  },
  methods: {
    handleUploadComplete({ file, response }) {
      console.log("Upload completed:", file, response);
    },
    handleUploadError({ file, error }) {
      console.error("Upload failed:", file, error);
    },
  },
};
</script>
