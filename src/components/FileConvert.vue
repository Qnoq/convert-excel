<template>
  <div class="flex items-center justify-center min-h-screen bg-gray-100">
    <Card class="w-full max-w-md">
      <CardHeader>
        <CardTitle>Convertisseur de fichiers</CardTitle>
        <CardDescription>Convertissez vos fichiers TXT, XLSX/CSV, ou TRA</CardDescription>
      </CardHeader>
      <CardContent>
        <div class="space-y-4">
          <div class="grid w-full max-w-sm items-center gap-1.5">
            <Label for="file-upload">Fichiers</Label>
            <Input id="file-upload" type="file" @change="handleFileUpload" multiple />
          </div>
          <div class="grid w-full max-w-sm items-center gap-1.5">
            <Label for="conversion-type">Type de conversion</Label>
            <Select v-model="conversionType">
              <SelectTrigger id="conversion-type">
                <SelectValue :placeholder="conversionType" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="txtToXlsx">TXT vers XLSX/CSV</SelectItem>
                <SelectItem value="xlsxToTxt">XLSX vers TXT</SelectItem>
                <SelectItem value="xlsxToTra">XLSX vers TRA</SelectItem>
              </SelectContent>
            </Select>
          </div>
          <div v-if="conversionType === 'txtToXlsx'" class="grid w-full max-w-sm items-center gap-1.5">
            <Label for="output-format">Format de sortie</Label>
            <Select v-model="outputFormat">
              <SelectTrigger id="output-format">
                <SelectValue :placeholder="outputFormat" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="xlsx">XLSX</SelectItem>
                <SelectItem value="csv">CSV</SelectItem>
              </SelectContent>
            </Select>
          </div>
        </div>
      </CardContent>
      <CardFooter>
        <Button @click="convertFile" :disabled="files.length === 0">
          Convertir
        </Button>
      </CardFooter>
    </Card>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import * as XLSX from 'xlsx';
import JSZip from 'jszip';
import { saveAs } from 'file-saver';
import { Button } from '@/components/ui/button';
import { Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter } from '@/components/ui/card';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';

const files = ref<File[]>([]);
const outputFormat = ref<'xlsx' | 'csv'>('xlsx');
const conversionType = ref<'txtToXlsx' | 'xlsxToTxt' | 'xlsxToTra'>('txtToXlsx');

const handleFileUpload = (event: Event): void => {
  const target = event.target as HTMLInputElement;
  if (target.files) {
    files.value = Array.from(target.files);
  }
};

const convertFile = async (): Promise<void> => {
  if (files.value.length === 0) {
    alert('Veuillez sélectionner au moins un fichier');
    return;
  }

  if (conversionType.value === 'txtToXlsx') {
    await convertTxtToXlsx();
  } else if (conversionType.value === 'xlsxToTxt') {
    await convertXlsxToTxt();
  } else if (conversionType.value === 'xlsxToTra') {
    await convertXlsxToTra();
  }
};

const convertTxtToXlsx = async (): Promise<void> => {
  const zip = new JSZip();

  for (const file of files.value) {
    const content = await readFileContent(file);
    const lines = content.split('\n');
    const data = lines.map(line => line.split('|').map(cell => cell.trim()));

    const wb = XLSX.utils.book_new();
    const ws = XLSX.utils.aoa_to_sheet(data);
    XLSX.utils.book_append_sheet(wb, ws, "Feuille1");

    const fileExtension = outputFormat.value;
    const fileName = `${file.name.replace('.txt', '')}.${fileExtension}`;
    const fileContent = XLSX.write(wb, { bookType: fileExtension, type: 'array' });

    zip.file(fileName, fileContent);
  }

  const zipContent = await zip.generateAsync({ type: 'blob' });
  saveAs(zipContent, 'fichiers_convertis.zip');
};

const convertXlsxToTxt = async (): Promise<void> => {
  const zip = new JSZip();

  for (const file of files.value) {
    const arrayBuffer = await file.arrayBuffer();
    const workbook = XLSX.read(arrayBuffer, { type: 'array' });
    const sheetName = workbook.SheetNames[0];
    const worksheet = workbook.Sheets[sheetName];
    const data = XLSX.utils.sheet_to_json(worksheet, { header: 1 }) as string[][];

    const textContent = data.map(row => row.join('|')).join('\n');
    const fileName = `${file.name.replace(/\.(xlsx|xls)$/, '')}.txt`;

    zip.file(fileName, textContent);
  }

  const zipContent = await zip.generateAsync({ type: 'blob' });
  saveAs(zipContent, 'fichiers_convertis_txt.zip');
};

const convertXlsxToTra = async (): Promise<void> => {
  const zip = new JSZip();

  for (const file of files.value) {
    const arrayBuffer = await file.arrayBuffer();
    const workbook = XLSX.read(arrayBuffer, { type: 'array' });
    const sheetName = workbook.SheetNames[0];
    const worksheet = workbook.Sheets[sheetName];
    const data = XLSX.utils.sheet_to_json(worksheet, { header: 1 }) as string[][];

    // Convert to TRA format (this is a placeholder, adjust according to Cegid's TRA specifications)
    const traContent = data.map(row => row.join('|')).join('\n');
    const fileName = `${file.name.replace(/\.(xlsx|xls)$/, '')}.tra`;

    zip.file(fileName, traContent);
  }

  const zipContent = await zip.generateAsync({ type: 'blob' });
  saveAs(zipContent, 'fichiers_convertis_tra.zip');
};

const readFileContent = (file: File): Promise<string> => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = (e) => {
      if (e.target && typeof e.target.result === 'string') {
        resolve(e.target.result);
      } else {
        reject(new Error('Échec de la lecture du fichier'));
      }
    };
    reader.onerror = (e) => reject(e);
    reader.readAsText(file);
  });
};
</script>