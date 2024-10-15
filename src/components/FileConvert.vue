<template>
    <div>
        <input type="file" @change="handleFileUpload" multiple />
        <select v-model="outputFormat">
        <option value="xlsx">XLSX</option>
        <option value="csv">CSV</option>
      </select>
      <button @click="convertFile">Convertir</button>
    </div>
  </template>
  
  <script setup>
  import { ref } from 'vue';
  import * as XLSX from 'xlsx';
  import JSZip from 'jszip';
  import { saveAs } from 'file-saver';

  const files = ref([]);
  const file = ref(null);
  const outputFormat = ref('xlsx');
  
  const handleFileUpload = (event) => {
    files.value = Array.from(event.target.files);
    };

    const convertFile = async () => {
    if (files.value.length === 0) {
        alert('Veuillez sélectionner au moins un fichier');
        return;
    }

    const zip = new JSZip();

    for (const file of files.value) {
        const content = await readFileContent(file);
        const lines = content.split('\n');
        const data = lines.map(line => line.split('|').map(cell => cell.trim()));

        const wb = XLSX.utils.book_new();
        const ws = XLSX.utils.aoa_to_sheet(data);
        XLSX.utils.book_append_sheet(wb, ws, "Feuille1");

        const fileExtension = outputFormat.value === 'xlsx' ? 'xlsx' : 'csv';
        const fileName = `${file.name.replace('.txt', '')}.${fileExtension}`;
        const fileContent = XLSX.write(wb, { bookType: fileExtension, type: 'array' });

        zip.file(fileName, fileContent);
    }

    const zipContent = await zip.generateAsync({ type: 'blob' });
    saveAs(zipContent, 'fichiers_convertis.zip');
    };

    const readFileContent = (file) => {
    return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.onload = (e) => resolve(e.target.result);
        reader.onerror = (e) => reject(e);
        reader.readAsText(file);
    });
    };
  </script>