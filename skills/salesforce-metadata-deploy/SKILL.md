---
name: salesforce-metadata-deploy
description: Panduan dan otomatisasi deployment SFDX source metadata (Objects, Fields, Flows, Layouts, Permission Sets, dll.) ke Salesforce org menggunakan Salesforce CLI (`sf project deploy start`), dilengkapi dry-run/validasi, multi-pass dependency resolution, dan diagnosa error cerdas.
---

# Salesforce Metadata Deploy

Gunakan skill ini ketika diminta mendeploy komponen metadata SFDX lokal ke Salesforce org target, memvalidasi deployment, atau mendiagnosa dan memperbaiki kegagalan deploy metadata.

## Prinsip & Batasan Utama
1. **Verifikasi Target Org**: Selalu cek dan konfirmasi alias/username target org (`sf config get target-org` atau via parameter) sebelum menjalankan deployment nyata.
2. **Prioritaskan Dry-Run**: Gunakan flag `--dry-run` terlebih dahulu untuk memvalidasi dependency dan kompatibilitas metadata sebelum menulis perubahan ke target org.
3. **Multi-Pass Otomatis untuk Urutan Dependency**: Jika kegagalan hanya disebabkan oleh urutan deploy (misal field custom belum ada saat layout atau validation rule di-deploy), kelompokkan dan deploy dalam beberapa tahap secara otomatis.
4. **Approval untuk Perubahan Destruktif**: Jika error memerlukan modifikasi file metadata (menghapus referensi field yang hilang, mengubah XML, atau exclude komponen), rangkum akar masalah dan minta konfirmasi sebelum mengubah file.
5. **No Apex Test Class Auto-Gen**: Jangan membuat Apex test class atau Jest test untuk demonstrasi konfigurasi/setup presales.

---

## Alur Kerja Deployment

### Langkah 1: Pengecekan Environment & Target Org
Jalankan pengecekan direktori project dan default org:
```bash
sf config get target-org
```
Tampilkan informasi ringkas ke pengguna:
- **Project path**: direktori kerja saat ini
- **Target Org**: alias atau username Salesforce target
- **Komponen**: path direktori (`--source-dir`) atau daftar metadata (`--metadata`) yang akan di-deploy

Tanyakan/konfirmasi jika target org belum sesuai atau belum diatur.

---

### Langkah 2: Validasi / Dry-Run
Jalankan validasi tanpa menerapkan perubahan:
```bash
sf project deploy validate --source-dir <path> --target-org <alias/username>
# atau
sf project deploy start --dry-run --source-dir <path> --target-org <alias/username>
```

- Jika **sukses** (green): Lanjut ke konfirmasi eksekusi deploy nyata.
- Jika **gagal**: Analisis output error JSON atau log CLI (`sf project deploy report`).

---

### Langkah 3: Penanganan Error & Klasifikasi Masalah

Ketika deploy/validasi gagal, klasifikasikan error ke salah satu dari dua kategori:

#### Kategori A: Masalah Urutan Dependency (Multi-Pass Otomatis)
Contoh kasus:
- Field custom belum ter-deploy tetapi sudah direferensikan di Page Layout, Validation Rule, atau Flow.
- Custom Object belum ada saat Child Object/Master-Detail Custom Field di-deploy.
- Record Type mereferensikan Picklist Value yang baru dibuat.

**Tindakan Otomatis**:
1. Buat urutan deploy bertahap (Pass 1 -> Pass 2 -> Pass 3):
   - **Pass 1 (Struktur Dasar)**: `CustomObject`, `CustomField`, `GlobalPicklist`
   - **Pass 2 (Logika & Aturan)**: `ValidationRule`, `RecordType`, `BusinessProcess`
   - **Pass 3 (Antarmuka & Keamanan)**: `Layout`, `FlexiPage`, `Flow`, `PermissionSet`, `Profile`
2. Eksekusi per pass dengan flag `--metadata` atau `--source-dir` terpisah.
3. Laporkan hasil tiap pass.

#### Kategori B: Referensi Hilang / Metadata Rusak (Diagnosa & Rekomendasi)
Contoh kasus:
- Komponen mereferensikan field/objek yang tidak ada di metadata lokal maupun di target org (missing dependency).
- Profile/PermissionSet mereferensikan komponen standard org yang belum diaktifkan (misal chatter, knowledge).
- Versi API metadata tidak didukung oleh target org.

**Tindakan**:
1. Bedah log error dan ekstrak:
   - Nama komponen bermasalah (`Component Name`)
   - Pesan error teknis (`Problem`)
   - Lokasi file XML / JSON (`File Path`)
2. Rangkum akar masalah dalam 2-3 kalimat berbahasa Indonesia sederhana.
3. Berikan opsi tindakan konkrit:
   - **Opsi 1**: Hapus baris referensi hilang dari XML/file lokal (tunjukkan diff yang diusulkan).
   - **Opsi 2**: Exclude komponen bermasalah dari daftar deploy (misal deploy tanpa PermissionSet/Layout tersebut).
   - **Opsi 3**: Tambahkan dependensi yang hilang dari target org (`sf project retrieve start`).
4. **Tunggu persetujuan pengguna** sebelum melakukan perubahan file atau exclude permanen.

---

### Langkah 4: Eksekusi Deploy Nyata
Setelah validasi lolos atau perbaikan disetujui:
```bash
sf project deploy start --source-dir <path> --target-org <alias/username>
```
Gunakan flag opsional bila relevan:
- `--ignore-conflicts`: Jika ingin menimpa metadata remote di scratch org/sandbox.
- `--concise`: Untuk output yang lebih ringkas.

---

### Langkah 5: Pelaporan Status Akhir
Laporkan hasil deployment secara transparan:
- Komponen yang berhasil di-deploy (`Deployed`)
- Komponen yang dilewati/di-exclude (jika ada)
- Status akhir target org (URL/alias target org yang berhasil diperbarui)
