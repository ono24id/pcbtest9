# pcbtest9

[English](README.md) | **Bahasa Indonesia**

Proyek perangkat keras KiCad 10.

> ⚠️ **Penafian Kode Hasil AI:** Proyek ini berisi kode yang dibuat dengan bantuan perangkat AI. Meskipun fungsi utamanya telah diuji dan divalidasi secara menyeluruh, harap tinjau seluruh kode sebelum digunakan di lingkungan produksi.

**Daftar isi**
- [Menggunakan templat ini](#menggunakan-templat-ini)
- [Struktur direktori](#struktur-direktori)
- [Menambahkan komponen dan pustaka](#menambahkan-komponen-dan-pustaka)
  - [A. Pustaka bawaan KiCad](#a-pustaka-bawaan-kicad)
  - [B. Repositori git pustaka KiCad (submodule)](#b-repositori-git-pustaka-kicad-submodule)
  - [C. Berkas unduhan dari vendor (zip)](#c-berkas-unduhan-dari-vendor-zip)
  - [D. Membuat sendiri](#d-membuat-sendiri)
  - [Setelah menambahkan komponen](#setelah-menambahkan-komponen)
- [Asisten AI (MCP)](#asisten-ai-mcp)
- [CI dan rilis](#ci-dan-rilis)

## Menggunakan templat ini

1. Klik **Use this template → Create a new repository** di GitHub. Nama repositori
   akan menjadi nama proyek KiCad (misalnya `sensor-board`).
2. Workflow **KiCad** berjalan satu kali:
   - job **Rename project** mengganti `pcbtest9` dengan nama repositori, baik pada nama berkas
     maupun isi berkas (`sources/sensor-board/sensor-board.kicad_pro`, `libraries/sensor-board.kicad_sym`,
     dan seterusnya), membuat ulang UUID skematik utama, lalu menyimpan perubahan sebagai commit
     `github-actions[bot]`;
   - job **Build** menjalankan ERC, DRC, dan membuat berkas fabrikasi untuk proyek yang namanya
     telah diganti.
3. Jalankan `git pull`, lalu buka berkas `.kicad_pro` di KiCad.

Setelah itu, setiap *push* hanya menghasilkan satu *run* berjudul *"Build: …"* (job **Rename project**
dilewati).

Jika repositori diklon secara lokal tanpa GitHub, jalankan skrip berikut secara manual:

```bash
scripts/init.sh nama-proyek   # tanpa argumen: menggunakan nama direktori repositori
```

> Apabila commit penggantian nama ditolak, buka **Settings → Actions → General → Workflow permissions**,
> pilih **Read and write permissions**, lalu jalankan workflow secara manual
> (tab Actions → KiCad → Run workflow).
<!-- template:end -->

## Struktur direktori

```
sources/pcbtest9/        proyek KiCad (.kicad_pro/.kicad_sch/.kicad_pcb) + tabel pustaka
libraries/
  pcbtest9.kicad_sym     simbol khusus proyek
  pcbtest9.pretty/       footprint khusus proyek
  pcbtest9.3dshapes/     model 3D (STEP/WRL)
  external/<nama>/           pustaka eksternal (git submodule)
scripts/                     init.sh, add/remove-library.sh, mcp-kicad.sh
.mcp.json, .vscode/, .cursor/ konfigurasi MCP untuk asisten AI
AGENTS.md, CLAUDE.md         petunjuk untuk agen AI
.github/workflows/kicad.yml  ERC, DRC, dan berkas fabrikasi
```

Pustaka (*library*) proyek telah terdaftar di `sym-lib-table` dan `fp-lib-table` proyek dengan jalur
`${KIPRJMOD}/../../libraries/...`, sehingga tetap berfungsi di mana pun repositori diklon.
Untuk model 3D, isi jalur pada footprint dengan `${KIPRJMOD}/../../libraries/pcbtest9.3dshapes/<berkas>.step`.

Blok judul (*title block*) menggunakan variabel teks `${PROJECT}`, `${REVISION}`, dan `${CURRENT_DATE}`.
Nilai `REVISION` di KiCad adalah `dev` dan diisi otomatis oleh CI berdasarkan tag atau commit git.

## Menambahkan komponen dan pustaka

Pertama, tentukan **dari mana komponen tersebut berasal**. Jawabannya menentukan langkah yang digunakan.

| Komponen tersedia… | Langkah | Lokasi penyimpanan |
| --- | --- | --- |
| di pustaka bawaan KiCad | **A.** langsung digunakan | – |
| di repositori git pustaka KiCad | **B.** `scripts/add-library.sh <url>` | `libraries/external/<nama>/` (submodule) |
| dalam berkas unduhan vendor (SnapEDA, Ultra Librarian, zip dari Mouser/DigiKey) | **C.** diimpor | `libraries/pcbtest9.*` |
| tidak tersedia di mana pun | **D.** dibuat sendiri | `libraries/pcbtest9.*` |

### A. Pustaka bawaan KiCad

Selalu periksa pustaka bawaan terlebih dahulu: resistor, kapasitor, LED, *pin header*, regulator umum,
*header* 40 pin Raspberry Pi, dan sebagainya. Tekan **A** di editor skematik, lalu cari komponennya.
Pustaka bawaan sudah terpasang bersama KiCad (termasuk di *container* CI), sehingga tidak perlu didaftarkan.

### B. Repositori git pustaka KiCad (submodule)

```bash
scripts/add-library.sh https://github.com/<owner>/<kicad-lib>.git          # -> libraries/external/<kicad-lib>
scripts/add-library.sh https://github.com/<owner>/<kicad-lib>.git mylib -b main
git commit -m "Add mylib library"
```

Skrip ini menjalankan `git submodule add`, lalu mendaftarkan setiap berkas `*.kicad_sym` dan direktori
`*.pretty` di dalam submodule ke `sym-lib-table` dan `fp-lib-table` proyek (nama panggilan = nama berkas,
jalur melalui `${KIPRJMOD}`). Nama panggilan yang sudah ada akan dilewati. Setelah itu, buka kembali
proyek di KiCad.

Submodule menjaga ukuran repositori tetap kecil, mengunci versi pustaka untuk setiap revisi papan, dan
dapat diperbarui apabila vendor melakukan perbaikan. **Jangan pernah mengubah berkas di
`libraries/external/`**; apabila perlu mengubah suatu komponen, salin komponen tersebut ke pustaka
proyek (langkah C/D).

```bash
git clone --recursive <repo-url>              # mengklon repositori beserta pustakanya
git submodule update --init --recursive       # setelah clone atau pull biasa
git submodule update --remote libraries/external/mylib # memperbarui pustaka ke commit terbaru
```

Untuk menghapus pustaka:

```bash
scripts/remove-library.sh mylib
git commit -m "Remove mylib library"
```

Skrip ini melepas dan menghapus submodule (termasuk salinannya di `.git/modules`) serta menghapus
entrinya dari tabel pustaka. Skrip akan menolak berjalan selama skematik atau papan masih menggunakan
simbol atau footprint dari pustaka tersebut; ganti komponen tersebut terlebih dahulu, atau gunakan `--force`.

Gunakan URL `https://` agar CI dapat mengambil pustaka tersebut. Untuk repositori pustaka privat,
tambahkan *repository secret* `SUBMODULE_TOKEN` (PAT dengan akses baca); proses *checkout* di CI akan
menggunakannya secara otomatis.

### C. Berkas unduhan dari vendor (zip)

Impor berkas tersebut ke pustaka milik proyek, yang sudah terdaftar:

| Berkas | Simpan di | Cara |
| --- | --- | --- |
| Simbol (`.kicad_sym`) | `libraries/pcbtest9.kicad_sym` | Symbol Editor → pilih pustaka `pcbtest9` → **File → Import Symbol** |
| Footprint (`.kicad_mod`) | `libraries/pcbtest9.pretty/` | salin berkasnya, atau Footprint Editor → **File → Import Footprint** |
| Model 3D (`.step`) | `libraries/pcbtest9.3dshapes/` | salin berkasnya; pada **Properties → 3D Models** footprint, isi `${KIPRJMOD}/../../libraries/pcbtest9.3dshapes/<berkas>.step` |

Setelah itu, isi kolom **Footprint** pada simbol dengan `pcbtest9:<footprint>`.

> Selalu periksa footprint hasil unduhan terhadap *datasheet* (ukuran *pad*, jarak antarpin, dan posisi
> pin 1). Kesalahan footprint baru akan diketahui setelah papan tiba dari pabrik.

### D. Membuat sendiri

1. Symbol Editor → **New Symbol** di pustaka `pcbtest9`. Tentukan **jenis elektrik** setiap pin
   (Input, Output, Power input, dan sebagainya) dengan benar, karena ERC menggunakannya.
2. Footprint Editor → **New Footprint** di `pcbtest9.pretty`, sesuai *recommended land pattern*
   pada *datasheet* (atau gunakan **Footprint Wizard** untuk kemasan standar seperti QFN atau SOIC).
3. Isi kolom **Footprint** pada simbol dan, jika tersedia, tambahkan model 3D seperti pada langkah C.

### Setelah menambahkan komponen

1. Tempatkan komponen di skematik, lalu jalankan **Update PCB from Schematic** (F8).
2. Jalankan ERC dan DRC (atau minta bantuan asisten AI).
3. Jalankan `git push`. Apabila CI gagal padahal pemeriksaan di komputer lokal berhasil, biasanya jalur
   pustaka masih absolut (`/Users/...`) dan belum relatif melalui `${KIPRJMOD}`.

## Asisten AI (MCP)

Repositori ini telah menyertakan server [Model Context Protocol](https://modelcontextprotocol.io)
untuk KiCad, yaitu [kicad-mcp-pro](https://github.com/oaslananka/kicad-mcp-pro), yang langsung
terhubung ke proyek ini:

| Klien | Konfigurasi |
| --- | --- |
| Claude Code | `.mcp.json` (setujui server `kicad` saat pertama kali dijalankan) |
| VS Code / Copilot | `.vscode/mcp.json` |
| Cursor | `.cursor/mcp.json` |
| Klien lain | perintah `bash scripts/mcp-kicad.sh` (stdio) |

Kebutuhan: [uv](https://docs.astral.sh/uv/getting-started/installation/) (`uvx`) dan KiCad 10
dengan `kicad-cli` pada `PATH`. Di Windows, jalankan melalui Git Bash atau WSL.
Server menemukan `sources/*/*.kicad_pro` secara otomatis, sehingga tidak ada pengaturan yang perlu
diubah untuk setiap proyek.

Perilaku server dapat diatur melalui variabel lingkungan yang dibaca oleh `scripts/mcp-kicad.sh`:
`KICAD_MCP_OPERATING_MODE` (`readonly`, `write` *(bawaan)*, `manufacturing`),
`KICAD_MCP_PROFILE` (`default`, `review`, `build`, `release`, `full`, dan lainnya), serta
`KICAD_MCP_PACKAGE` (misalnya `kicad-mcp-pro==3.35.0` untuk mengunci versi).

Aturan proyek untuk agen AI terdapat di [`AGENTS.md`](AGENTS.md) (diimpor oleh `CLAUDE.md`).

## CI dan rilis

Setiap *push* dan *pull request* menjalankan [`kicad.yml`](.github/workflows/kicad.yml) di dalam
*container* `kicad/kicad:10.0`:

| Tahap | Keluaran |
| --- | --- |
| ERC, DRC (+ kesesuaian skematik dan PCB) | `reports/erc.rpt`, `reports/drc.rpt` — job gagal apabila terdapat *error* |
| Skematik | `pcbtest9-schematic.pdf`, `pcbtest9-bom.csv` |
| PCB | `gerbers/` + `pcbtest9-gerbers.zip`, berkas bor + peta bor, `pcbtest9-pos.csv`, `pcbtest9-pcb.pdf`, `pcbtest9.step` |

Keluaran dapat diunduh dari tab **Actions** (bagian *artifact*). Lapisan Gerber mengikuti pengaturan
**File → Plot** yang tersimpan di berkas papan.

Untuk rilis produksi:

```bash
git tag v1.0 && git push origin v1.0
```

Apabila ERC dan DRC bersih, GitHub Release `v1.0` akan dibuat dan berisi arsip zip lengkap, Gerber,
skematik, BOM, serta berkas posisi komponen.
