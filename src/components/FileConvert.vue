<template>
  <div class="flex items-center justify-center min-h-screen bg-gray-100">
    <Card class="w-full max-w-md">
      <CardHeader>
        <CardTitle>Convertisseur de fichiers FEC</CardTitle>
        <CardDescription>Ajustez les dates dans vos fichiers FEC selon la période fiscale</CardDescription>
      </CardHeader>
      <CardContent>
        <div class="space-y-4">
          <div class="grid w-full max-w-sm items-center gap-1.5">
            <Label for="file-upload">Fichier FEC</Label>
            <Input id="file-upload" type="file" @change="handleFileUpload" />
          </div>
          <div class="grid w-full max-w-sm items-center gap-1.5">
            <Label for="fiscal-period">Période fiscale</Label>
            <Select v-model="selectedPeriod">
              <SelectTrigger class="w-full">
                <SelectValue placeholder="Sélectionnez une période fiscale" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="tds">TDS (01/04 au 31/03)</SelectItem>
                <SelectItem value="4m">4M et 2&2M (01/01 au 31/12)</SelectItem>
                <SelectItem value="mattei">MATTEÏ (01/07 au 30/06)</SelectItem>
              </SelectContent>
            </Select>
          </div>
          <div class="grid w-full max-w-sm items-center gap-1.5">
            <Label for="fiscal-year">Année fiscale de début</Label>
            <Input 
              id="fiscal-year" 
              v-model="fiscalYear" 
              type="number" 
              placeholder="Ex: 2023"
              @input="handleFiscalYearInput"
            />
          </div>
        </div>
      </CardContent>
      <CardFooter>
        <Button @click="processFile" :disabled="!file || !selectedPeriod || !fiscalYear">
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
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';

const file = ref<File | null>(null);
const selectedPeriod = ref<string>('');
const fiscalYear = ref<string>('');

const fiscalPeriods = {
  tds: { start: '0401', end: '0331' },
  '4m': { start: '0101', end: '1231' },
  mattei: { start: '0701', end: '0630' }
};

const handleFileUpload = (event: Event): void => {
  const target = event.target as HTMLInputElement;
  if (target.files && target.files.length > 0) {
    file.value = target.files[0];
  }
};

const handleFiscalYearInput = (event: Event): void => {
  const target = event.target as HTMLInputElement;
  fiscalYear.value = target.value;
};

const processFile = async (): Promise<void> => {
  if (!file.value || !selectedPeriod.value || !fiscalYear.value) {
    alert('Veuillez sélectionner un fichier, une période fiscale et spécifier l\'année fiscale');
    return;
  }

  const content = await readFileContent(file.value);
  const lines = content.split('\n');
  const header = lines[0];
  const adjustedLines = lines.slice(1).map(line => adjustLine(line));

  const adjustedContent = [header, ...adjustedLines].join('\n');
  const blob = new Blob([adjustedContent], { type: 'text/plain;charset=utf-8' });
  saveAs(blob, `${file.value.name.replace('.fec', '')}_adjusted`);
};

const adjustLine = (line: string): string => {
  const fields = line.split('|');
  if (fields.length > 3) {
    const dateIndex = 3; // L'index de la colonne EcritureDate
    const date = fields[dateIndex];
    fields[dateIndex] = adjustDate(date);
  }
  return fields.join('|');
};

const adjustDate = (dateStr: string): string => {
  if (!fiscalYear.value) return dateStr;

  const year = parseInt(fiscalYear.value, 10);
  if (isNaN(year)) return dateStr;

  const { start, end } = fiscalPeriods[selectedPeriod.value as keyof typeof fiscalPeriods];
  const startYear = year;
  const endYear = start <= end ? startYear : startYear + 1;

  const fiscalStart = `${startYear}${start}`;
  const fiscalEnd = `${endYear}${end}`;

  if (dateStr < fiscalStart) {
    return fiscalStart;
  } else if (dateStr > fiscalEnd) {
    return fiscalEnd;
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