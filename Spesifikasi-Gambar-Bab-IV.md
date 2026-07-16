# Spesifikasi Gambar Bab IV — Perancangan

Dokumen pegangan untuk menggambar 9 gambar Bab IV. Setiap gambar di `Bab IV - Perancangan.tex` saat ini masih **placeholder**; spesifikasi yang sama juga tertulis sebagai komentar `% SPEC GAMBAR …` di berkas `.tex`.

## Konvensi umum
- **Alat**: draw.io (diagrams.net) atau Excalidraw. Ekspor **PNG** (resolusi tinggi, mis. 2×).
- **Legenda warna** (konsisten di seluruh bab):
  - 🔴 Merah = faktor/aliran yang **menaikkan** risiko.
  - 🟢 Hijau = faktor/aliran yang **menurunkan** risiko.
  - (opsional) kotak **coral** = komponen/tahap baru, kotak **abu** = tahap konsep lama — mengikuti gaya Bab V bila ingin selaras.
- **Simpan** ke folder `images/bab4/` (buat folder ini bila belum ada).
- **Cara memasang** di LaTeX — ganti baris placeholder:
  ```latex
  \bivplaceholder{...}
  ```
  menjadi:
  ```latex
  \includegraphics[width=0.8\textwidth]{images/bab4/arsitektur_final.png}
  ```
- **Larangan global** (konsistensi versi final): **tanpa** RAG / Vector Database / Retriever / Embedding / OpenAI; LLM = **Groq `llama-3.1-8b-instant`**; input **8 fitur non-invasif** (tanpa tekanan darah, riwayat keluarga, konsumsi garam).

## Checklist
- [ ] 4.1 Arsitektur Sistem Final
- [ ] 4.1b Alur Kerja End-to-End (runtime)
- [ ] 4.2 Alur Pra-pemrosesan Data
- [ ] 4.3 Alur Modul Prediksi
- [ ] 4.4 Alur Modul XAI
- [ ] 4.5 Arsitektur Narasi Dua Lapisan
- [ ] 4.6 Wireframe Formulir 3 Langkah
- [ ] 4.7 Wireframe Halaman Hasil
- [ ] 4.8 Alur As-Is vs To-Be

---

## 4.1 — Arsitektur Sistem Final
- **Label**: `fig:arsitektur-final` · **Ganti**: `arsitektur sistem.png` · **Saran berkas**: `arsitektur_final.png`
- **Diturunkan dari**: FR01–FR05 (arsitektur menyeluruh)
- **Jenis**: diagram blok berlapis (top-down), 4 bagian.

**Komponen & aliran:**
1. **Client Layer** — kotak "Aplikasi Web (Next.js / React, TypeScript)" berisi *Formulir 3 langkah* + *Halaman Hasil*.
   - Panah dua arah ke backend berlabel **"REST/JSON via HTTPS — POST /api/v1/analyze"**.
2. **Application Layer (Backend FastAPI)** — rangkaian **sekuensial**:
   `[Preprocessing]` → `[Inference: XGBoost predict_proba]` → `[Ambang Youden 0,516 → kelas Risiko Tinggi/Rendah]` → `[XAI: SHAP TreeExplainer → atribusi fitur]` → `[Modul Narasi: Lapisan Deterministik → LLM (penghalus)]` → `[Respons JSON tunggal: status + grafik SHAP + narasi]`.
3. **Model & Artefak** — kotak berkas `joblib` {model XGBoost + ambang Youden + urutan fitur} dan berkas nilai imputasi cadangan; panah ke `Inference`.
4. **External Service** — kotak **"LLM Groq — llama-3.1-8b-instant"**; panah "materi saran (teks) → narasi halus"; catatan kecil **"fallback deterministik bila LLM tidak tersedia"**.

**Wajib**: TIDAK ADA Vector DB / Retriever / RAG / basis data relasional.

---

## 4.1b — Alur Kerja End-to-End (runtime)
- **Label**: `fig:alur-runtime` · **Ganti**: `alur kerja sistem.png` · **Saran berkas**: `alur_runtime.png`
- **Diturunkan dari**: FR01–FR04 (alur pemakaian)
- **Jenis**: alur linear (kiri→kanan) 5 langkah.

**Aliran:**
`[Pengguna: input 8 fitur non-invasif]` → `[XGBoost → probabilitas risiko]` → `[Ambang Youden → "Risiko Tinggi"/"Risiko Rendah"]` → `[SHAP TreeExplainer → kontribusi fitur (arah & besar)]` → `[Narasi: deterministik menetapkan angka/arah/fakta → LLM menghaluskan]` → `[Halaman Hasil: (1) status+meter, (2) grafik SHAP, (3) narasi+disclaimer]`.

---

## 4.2 — Alur Pra-pemrosesan Data
- **Label**: `fig:alur-preprocessing` · **Baru** · **Saran berkas**: `alur_preprocessing.png`
- **Diturunkan dari**: **FR01**
- **Jenis**: alur linear.

**Aliran:**
`[Masukan pengguna: usia, jenis kelamin, berat, tinggi, merokok, diabetes, kolesterol, kualitas & gangguan tidur]` → `[Validasi kelengkapan]` → `[Konversi berat+tinggi → IMT (di sisi klien)]` → `[Imputasi: median (numerik/ordinal), modus (biner) — parameter dari data latih]` → `[Encoding biner: is_female, is_smoker, has_diabetes, has_high_cholesterol]` → `[Vektor 8 fitur siap diprediksi]`.

**Anotasi**: "tekanan darah TIDAK disertakan (mencegah kebocoran data)".

---

## 4.3 — Alur Modul Prediksi
- **Label**: `fig:alur-prediksi` · **Baru** · **Saran berkas**: `alur_prediksi.png`
- **Diturunkan dari**: **FR02, NFR01, NFR02**
- **Jenis**: alur linear.

**Aliran:**
`[Vektor 8 fitur]` → `[Model XGBoost (class weighting saat latih)]` → `[predict_proba → skor probabilitas 0..1]` → `[Bandingkan dengan ambang Youden 0,516]` → `[Keputusan: "Risiko Tinggi" bila ≥ ambang, selain itu "Risiko Rendah"]`.

**Catatan**: keluaran probabilitas dipertahankan sehingga ambang dapat disetel ulang tanpa melatih ulang.

---

## 4.4 — Alur Modul XAI
- **Label**: `fig:alur-xai` · **Baru** · **Saran berkas**: `alur_xai.png`
- **Diturunkan dari**: **FR03, NFR04**
- **Jenis**: alur konvergen (dua masukan → satu proses).

**Aliran:**
`[Model XGBoost terlatih]` + `[Vektor fitur pasien]` → `[SHAP TreeExplainer]` → `[Vektor nilai SHAP bertanda per fitur (+ menaikkan / − menurunkan risiko)]` → `[Grafik kontribusi (bar horizontal): 🔴 menaikkan, 🟢 menurunkan]`.

**Catatan**: analisis global dalam satuan *log-odds*; analisis lokal dalam satuan *probabilitas*.

---

## 4.5 — Arsitektur Narasi Dua Lapisan
- **Label**: `fig:arsitektur-narasi` · **Baru** · **Saran berkas**: `arsitektur_narasi.png`
- **Diturunkan dari**: **FR04, FR05, NFR03**
- **Jenis**: diagram dua lapisan + jalur fallback.

**Aliran:**
`[Nilai SHAP + kondisi klinis pasien]`
→ **LAPISAN DETERMINISTIK** menetapkan: (a) arah dari tanda SHAP, (b) besaran = kontribusi lokal, (c) kategori magnitudo, (d) tier keandalan fitur, (e) pemicu saran dari kondisi klinis → menghasilkan **"materi saran (teks baku)"**
→ **LAPISAN LLM (Groq llama-3.1-8b-instant)**: **hanya menghaluskan** bahasa materi saran, dengan *guardrail* (persona, fakta terstruktur, larangan fabrikasi)
→ `[Narasi akhir]`.

**Tambahan**: panah putus-putus **"FALLBACK: bila LLM tak tersedia → pakai materi deterministik apa adanya"**.

---

## 4.6 — Wireframe Formulir 3 Langkah
- **Label**: `fig:wireframe-form` · **Baru** · **Saran berkas**: `wireframe_form.png`
- **Diturunkan dari**: **FR01**
- **Jenis**: wireframe UI, 3 kartu langkah + indikator progres.

**Isi:**
- **Langkah 1 — Demografi & Fisik**: usia, jenis kelamin, berat badan, tinggi badan.
- **Langkah 2 — Riwayat Medis & Gaya Hidup**: status merokok, riwayat diabetes, riwayat kolesterol tinggi.
- **Langkah 3 — Kualitas Tidur**: skor kualitas tidur (1–5), skor gangguan tidur (1–5).
- Tombol **Lanjut / Kembali / Analisis**.

**Pintasan**: boleh langsung memakai tangkapan layar purwarupa yang sudah ada — `images/prototype/form_input_1.png`, `_2.png`, `_3.png`.

---

## 4.7 — Wireframe Halaman Hasil
- **Label**: `fig:wireframe-hasil` · **Baru** · **Saran berkas**: `wireframe_hasil.png`
- **Diturunkan dari**: **FR02, FR03, FR04**
- **Jenis**: wireframe UI, 3 area vertikal.

**Isi:**
1. **Status risiko**: label "Risiko Tinggi/Rendah" + *meter* berwarna (hijau→merah) menampilkan skor probabilitas.
2. **Grafik kontribusi SHAP**: bar horizontal per faktor (🔴 menaikkan, 🟢 menurunkan, panjang = kontribusi relatif); sertakan nilai masukan pengguna.
3. **Narasi**: ringkasan risiko + 3 faktor teratas + paragraf saran + **disclaimer** "bukan pengganti diagnosis tenaga kesehatan".

**Pintasan**: boleh memakai tangkapan layar halaman hasil purwarupa bila tersedia.

---

## 4.8 — Alur As-Is vs To-Be
- **Label**: `fig:alur-diagnosis` · **Ganti**: `alursistemtobe.png` · **Saran berkas**: `as_is_to_be.png`
- **Diturunkan dari**: gap analysis Bab III
- **Jenis**: dua jalur berdampingan.

**As-Is (konvensional, reaktif):**
`[Pasien datang ke fasyankes]` → `[Pengukuran & anamnesis manual]` → `[Penilaian dokter]` → `[Intervensi reaktif]`.

**To-Be (skrining dini, transparan):**
`[Masyarakat awam isi data non-invasif di aplikasi]` → `[Prediksi + XAI + narasi]` → `[Pahami risiko & faktor penyebab]` → `[Tindak lanjut ke tenaga kesehatan bila berisiko]`.

**Catatan framing**: pengguna = masyarakat awam (skrining mandiri); tenaga kesehatan = penentu tindakan medis lanjutan. Hindari penggambaran dokter sebagai operator utama sistem.
