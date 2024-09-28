<template>
  <table>
    <thead>
      <tr>
        <th>Company Name</th>
        <th>Current</th>
        <th>1-30 days</th>
        <th>31-60 days</th>
        <th>61-90 days</th>
        <th>Over 90 days</th>
        <th>Total</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="company in companies" :key="company.customerID">
        <td>{{ company.customerID }}</td>
        <td>{{ company.current }}</td>
        <td>{{ company.oneToThirty }}</td>
        <td>{{ company.thirtyOneToSixty }}</td>
        <td>{{ company.sixtyOneToNinety }}</td>
        <td>{{ company.overNinety }}</td>
        <td>{{ company.total }}</td>
      </tr>
    </tbody>
  </table>
</template>

<script lang="ts" setup>
interface Invoice {
  countryCode: string;
  customerID: string;
  PaperlessDate: string;
  invoiceNumber: string;
  InvoiceDate: string;
  DueDate: string;
  InvoiceAmount: number;
  Disputed: string;
  SettledDate: string;
  PaperlessBill: string;
  DaysToSettle: number;
  DaysLate?: number; // added DaysLate as an optional field
}

interface Company {
  customerID: string;
  current: number;
  oneToThirty: number;
  thirtyOneToSixty: number;
  sixtyOneToNinety: number;
  overNinety: number;
  total: number;
}

import type { Ref } from "vue";
import { onMounted, ref } from "vue";

const companies: Ref<Company[]> = ref([]);

function readCSVFile(filePath: string) {
  return new Promise<string>((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    xhr.open("GET", filePath);
    xhr.onload = () => {
      if (xhr.status === 200) {
        resolve(xhr.responseText);
      } else {
        reject(xhr.statusText);
      }
    };
    xhr.onerror = () => {
      reject(xhr.statusText);
    };
    xhr.send();
  });
}

function convertCSVToInvoices(csv: string) {
  const lines = csv.split("\n");
  const headers = lines[0].split(",").map((header) => header.trim());
  const invoices = [];
  for (let i = 1; i < lines.length; i++) {
    const data = lines[i].split(",");
    if (data.length === headers.length) {
      const invoice: Invoice = {} as Invoice;
      for (let j = 0; j < headers.length; j++) {
        const value = data[j].trim();
        if (headers[j] === "InvoiceAmount") {
          invoice[headers[j]] = parseFloat(value);
        } else if (headers[j] === "DaysLate") {
          invoice["DaysLate"] = parseInt(value) || 0;
        } else {
          invoice[headers[j]] = value;
        }
      }
      invoices.push(invoice);
    }
  }
  return invoices;
}

function groupInvoices(invoices: Invoice[]) {
  return invoices.reduce((acc, curr) => {
    if (acc[curr.customerID]) {
      acc[curr.customerID].push(curr);
    } else {
      acc[curr.customerID] = [curr];
    }
    return acc;
  }, {} as { [key: string]: Invoice[] });
}

function calculateTotal(invoices: Invoice[]) {
  //round to 2 decimal places
  return (
    Math.round(
      invoices.reduce((acc, curr) => acc + curr.InvoiceAmount, 0) * 100
    ) / 100
  );
}

onMounted(() => {
  readCSVFile("/inputdata.csv").then((data) => {
    const invoices = convertCSVToInvoices(data) as Invoice[];
    const grouped = groupInvoices(invoices);
    companies.value = Object.keys(grouped).map((key) => {
      const invoices = grouped[key];
      const current = invoices.filter((invoice) => invoice.DaysLate! <= 0);
      const oneToThirty = invoices.filter(
        (invoice) => invoice.DaysLate! > 0 && invoice.DaysLate! <= 30
      );
      const thirtyOneToSixty = invoices.filter(
        (invoice) => invoice.DaysLate! > 30 && invoice.DaysLate! <= 60
      );
      const sixtyOneToNinety = invoices.filter(
        (invoice) => invoice.DaysLate! > 60 && invoice.DaysLate! <= 90
      );
      const overNinety = invoices.filter((invoice) => invoice.DaysLate! > 90);
      const total =
        Math.round(
          invoices.reduce((acc, curr) => acc + curr.InvoiceAmount, 0) * 100
        ) / 100;
      return {
        customerID: key,
        current: calculateTotal(current),
        oneToThirty: calculateTotal(oneToThirty),
        thirtyOneToSixty: calculateTotal(thirtyOneToSixty),
        sixtyOneToNinety: calculateTotal(sixtyOneToNinety),
        overNinety: calculateTotal(overNinety),
        total,
      };
    });
  });
});
</script>

<style lang="css">
table {
  border-collapse: collapse;
  width: 100%;
  color: #fff;
}

th,
td {
  text-align: left;
  padding: 8px;
  border: 1px solid #ddd;
}

th {
  background-color: #333;
  font-weight: bold;
}

tr:nth-child(even) {
  background-color: #555;
}

tr:hover {
  background-color: #444;
}

th:first-child,
td:first-child {
  border-left: none;
}

th:last-child,
td:last-child {
  border-right: none;
}
</style>
