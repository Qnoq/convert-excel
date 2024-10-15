<template>
    <div class="flex items-center justify-center min-h-screen bg-gray-100">
      <Card class="w-full max-w-md">
        <CardHeader>
          <CardTitle>Convertisseur de fichiers</CardTitle>
          <CardDescription>Convertissez vos fichiers TXT en XLSX ou CSV</CardDescription>
        </CardHeader>
        <CardContent>
          <div class="space-y-4">
            <div class="grid w-full max-w-sm items-center gap-1.5">
              <Label for="file-upload">Fichiers</Label>
              <Input id="file-upload" type="file" @change="handleFileUpload" multiple />
            </div>
            <div class="grid w-full max-w-sm items-center gap-1.5">
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