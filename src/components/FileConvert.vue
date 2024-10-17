<template>
  <div class="flex items-center justify-center min-h-screen bg-gray-100">
    <Card class="w-full max-w-md">
      <CardHeader>
        <CardTitle>Convertisseur de fichiers FEC</CardTitle>
        <CardDescription>Ajustez les dates dans vos fichiers FEC</CardDescription>
      </CardHeader>
      <CardContent>
        <div class="space-y-4">
          <div class="grid w-full max-w-sm items-center gap-1.5">
            <Label for="file-upload">Fichier FEC</Label>
            <Input id="file-upload" type="file" @change="handleFileUpload" />
          </div>
          <div class="grid w-full max-w-sm items-center gap-1.5">
            <Label for="fiscal-year">Année fiscale</Label>
            <Input id="fiscal-year" v-model="fiscalYear" type="number" placeholder="Ex: 2023" />
          </div>
        </div>
      </CardContent>
      <CardFooter>
        <Button @click="processFile" :disabled="!file || !fiscalYear">
          Traiter le fichier
        </Button>
      </CardFooter>
    </Card>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import { saveAs } from 'file-saver';
import { Button } from '@/components/ui/button';
import { Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter } from '@/components/ui/card';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';

const file = ref<File | null>(null);
const fiscalYear = ref<number | null>(null);

const handleFileUpload = (event: Event): void => {
  const target = event.target as HTMLInputElement;
  if (target.files && target.files.length > 0) {
    file.value = target.files[0];
  }
};

const processFile = async (): Promise<void> => {
  if (!file.value || !fiscalYear.value) {
    alert('Veuillez sélectionner un fichier et spécifier l\'année fiscale');
    return;
  }

  const content = await readFileContent(file.value);
  const lines = content.split('\n');
  const header = lines[0];
  const adjustedLines = lines.slice(1).map(line => adjustLine(line, fiscalYear.value!));

  const adjustedContent = [header, ...adjustedLines].join('\n');
  const blob = new Blob([adjustedContent], { type: 'text/plain;charset=utf-8' });
  saveAs(blob, `${file.value.name.replace('.fec', '')}_adjusted`);
};

const adjustLine = (line: string, fiscalYear: number): string => {
  const fields = line.split('|');
  if (fields.length > 3) {
    const dateIndex = 3; // L'index de la colonne EcritureDate
    const date = fields[dateIndex];
    fields[dateIndex] = adjustDate(date, fiscalYear);
  }
  return fields.join('|');
};

const adjustDate = (dateStr: string, fiscalYear: number): string => {
  const startDate = `${fiscalYear - 1}0401`;
  const endDate = `${fiscalYear}0331`;

  if (dateStr < startDate) {
    return startDate;
  } else if (dateStr > endDate) {
    return endDate;
  }

  return dateStr;
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