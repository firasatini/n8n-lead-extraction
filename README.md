# Leads from Email to Sheet — README

**Deskripsi singkat**

Workflow n8n ini mengambil email dari Gmail (dengan filter pengirim), mengekstrak informasi lead dari *snippet* email menggunakan node `Code` (JavaScript), lalu menambahkan (append) baris ke Google Sheets pada sheet bernama `Data`.

---

## Prasyarat

1. Akun n8n (self-hosted atau cloud) dengan akses untuk mengimpor workflow.
2. Kredensial Gmail OAuth2 yang terhubung di n8n.
3. Kredensial Google Sheets OAuth2 yang terhubung di n8n.
4. File workflow dalam format JSON (contoh: `Leads from Email to Sheet.json`).

---

## Cara impor workflow

1. Buka n8n → menu **Workflows** → **Import from File**.
2. Pilih file JSON workflow (misal `Leads from Email to Sheet.json`).
3. Setelah terimpor, pastikan untuk:

   * Menghubungkan kredensial Gmail dan Google Sheets yang sesuai ke node yang membutuhkan.
   * Mengubah documentId Google Sheets jika ingin menyimpan ke spreadsheet lain.
4. Aktifkan workflow (toggle `Active`) jika ingin dijalankan otomatis.

---

## Konfigurasi kredensial

* **Gmail Trigger** node membutuhkan kredensial Gmail OAuth2. Pastikan kredensial tersebut berizin membaca email dan men-set `pollTimes` sesuai kebutuhan.
* **Append row in sheet** node menggunakan kredensial Google Sheets OAuth2. Pastikan kredensial ini memiliki akses edit ke spreadsheet yang ditentukan.

> Di contoh ini, workflow default mengisi `documentId` dengan ID Google Sheet target. Jika Anda ingin menyimpan ke spreadsheet lain, ganti nilai `documentId` pada node `Append row in sheet`.

---

## Penjelasan node (runtut)

### 1) Gmail Trigger

* **Tipe:** `gmailTrigger`
* **Pengaturan penting:**

  * `pollTimes` menjalankan polling setiap hari pada jam tertentu (`hour: 3` di contoh).
  * `filters.sender` diset ke `Michelle from Dealls` — hanya email dari pengirim ini yang akan diproses.
* **Fungsi:** menangkap email baru sesuai filter dan meneruskan payload (termasuk `snippet`) ke node berikutnya.

### 2) Code (JavaScript)

* **Tipe:** `code` node — mengeksekusi JavaScript untuk mengekstrak field dari `snippet` email.
* **Kode yang digunakan (sama seperti di workflow):**

```js
const snippet = $input.all()[0].json.snippet;

const getValue = (label) => {
  const regex = new RegExp(label + ":\\s*([^\\n:]+?)(?=\\s+[a-z]+:|$)", "i");
  const match = snippet.match(regex);
  return match ? match[1].trim() : "";
};

return [
  {
    Nama: getValue("name"),
    Email: getValue("email"),
    Whatsapp: getValue("whatsapp"),
    Role: getValue("role"),
    Company: getValue("company"),
    Content: getValue("slug")
  }
];
```

* **Penjelasan singkat:**

  * Mengambil `snippet` (ringkasan teks email) dan mencari label seperti `name:`, `email:`, `whatsapp:`, `role:`, `company:`, `slug:`.
  * Regex digunakan untuk menangkap nilai setelah label sampai sebelum label berikutnya atau akhir teks.
  * Hasilnya berupa objek dengan kunci `Nama`, `Email`, `Whatsapp`, `Role`, `Company`, `Content`.

**Catatan:** Regex ini sensitif pada pola `label: value` dan akan gagal jika format email berbeda (misal `Name - John` atau `name = John`). Lihat bagian Troubleshooting untuk opsi perbaikan.

### 3) Append row in sheet (Google Sheets)

* **Tipe:** `googleSheets` dengan operasi `append`.

* **Pengaturan penting:**

  * `documentId`: ID spreadsheet tujuan (di contoh workflow sudah berisi ID).
  * `sheetName`: `Data`.
  * `columns.mappingMode`: `defineBelow`, memetakan `Nama`, `Email`, `Whatsapp`, `Role`, `Company`, `Content` ke kolom sheet.
  * `options.cellFormat`: `RAW` (tidak memformat otomatis).

* **Apa yang terjadi:** setiap output dari node `Code` akan menjadi satu baris baru di sheet `Data` dengan kolom sesuai mapping.

---

## Contoh input & output

### Contoh `snippet` email (yang diharapkan oleh parser):

```
name: John Doe
email: john@example.com
whatsapp: +62812xxxx
role: HR Manager
company: Contoh Co
slug: lead-from-landing
```

### Hasil di Google Sheets (baris baru):

| Nama     | Email                                       | Whatsapp   | Role       | Company   | Content           |
| -------- | ------------------------------------------- | ---------- | ---------- | --------- | ----------------- |
| John Doe | [john@example.com](mailto:john@example.com) | +62812xxxx | HR Manager | Contoh Co | lead-from-landing |

---

## Cara mengetes (manual)

1. Ubah `filters.sender` di node Gmail Trigger sementara ke alamat email Anda sendiri agar lebih cepat saat uji.
2. Kirim email percobaan ke akun Gmail yang terhubung dengan format `label: value` seperti contoh di atas.
3. Jalankan workflow secara manual atau tunggu polling berjalan pada jam yang diset.
4. Periksa Google Sheets apakah baris baru muncul.

---

## Troubleshooting & penyesuaian

1. **Field kosong** — kemungkinan format email tidak cocok dengan regex; periksa spacing, penulisan label (`name:` vs `Name:`) dan jangan gunakan `-` atau `=` sebagai pemisah.
2. **Pengirim tidak terdeteksi** — pastikan filter `sender` sama persis seperti pengirim pada header email. Anda bisa mengosongkan filter sementara untuk debugging.
3. **Akses Google Sheets ditolak** — cek apakah kredensial Google Sheets punya akses edit ke spreadsheet target.
4. **Regex terlalu ketat** — jika email memiliki label dalam bahasa lain (mis. `nama:`) atau format berbeda, sesuaikan `getValue` atau tambahkan beberapa variasi label:

```js
const getValue = (labelVariants) => {
  const labels = Array.isArray(labelVariants) ? labelVariants : [labelVariants];
  for (const label of labels) {
    const regex = new RegExp(label + ":\\s*([^\\n:]+?)(?=\\s+[a-z]+:|$)", "i");
    const match = snippet.match(regex);
    if (match) return match[1].trim();
  }
  return "";
};

// contoh pemanggilan: getValue(["name", "nama"])
```

5. **Multiple leads in one email** — saat ini kode mengambil `snippet` dari email (yang mungkin dipotong). Untuk multiple leads, lebih baik parse `raw` email body atau minta email dikirim dalam CSV/struktur terpisah.

---

## Keamanan & privasi

* Jangan bagikan kredensial OAuth2.
* Batasi akses Google Sheet hanya ke akun yang perlu.
* Pastikan kebijakan privasi perusahaan untuk menyimpan data kontak (GDPR/PDPA) terpenuhi.

---

## Kustomisasi yang umum

* Menambahkan validasi alamat email sebelum append (cek format dengan regex email).
* Menambahkan kolom `Timestamp` atau `Source` sebelum append.
* Mengirim notifikasi Slack/Telegram setelah row berhasil ditambahkan.

---

## Detail workflow (metadata)

* **Nama workflow:** Leads from Email to Sheet
* **ID workflow:** NsEOneN95y6jYCFP
* **Active:** true
* **Version ID:** b70a7162-18db-41f8-b7dc-4f03edd6f788

---

## Changelog

* v1.0 — README awal, menjelaskan cara impor, konfigurasi, dan contoh output.

---
