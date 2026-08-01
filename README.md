# Buku Usaha Kerupuk — Master, Produksi, Penjualan & Pembukuan

Aplikasi satu halaman (`index.html`) dengan 4 tab:

1. **Master Resep** — daftar jenis kerupuk (Puli, Kotak, dst), masing-masing dengan resep dan biaya sendiri untuk 1x skala produksi standar.
2. **Produksi** — catat tiap batch produksi nyata; jumlah kg bisa beda-beda dari standar, biaya resep otomatis diskalakan (misal produksi cuma setengah resep → semua biaya bahan ikut dibagi dua), tapi tetap bisa disesuaikan manual.
3. **Penjualan** — catat penjualan bertahap dari stok tiap batch produksi. Tiap kali disimpan, stok batch itu otomatis berkurang; kalau melebihi stok, sistem menolak.
4. **Pembukuan** — buku besar otomatis: gabungan semua biaya produksi (pengeluaran) dan penjualan (pemasukan), urut tanggal, dengan saldo berjalan, plus ringkasan total pendapatan/biaya/laba/stok belum terjual.

Tidak perlu build tool — tinggal upload dan buka.

## 1. Buat project Firebase (gratis)

1. Buka [console.firebase.google.com](https://console.firebase.google.com) → **Add project** → beri nama, misalnya `kerupuk-udang`.
2. Di sidebar kiri, klik **Build → Firestore Database** → **Create database** → pilih lokasi (misalnya `asia-southeast2`) → mulai dalam **test mode** dulu (nanti kita perketat aturannya di langkah 4).
3. Di sidebar kiri, klik ikon gerigi ⚙️ → **Project settings** → scroll ke **Your apps** → klik ikon `</>` (Web) → beri nickname → **Register app**.
4. Firebase akan menampilkan blok `firebaseConfig` seperti ini:

   ```js
   const firebaseConfig = {
     apiKey: "AIza...",
     authDomain: "kerupuk-udang.firebaseapp.com",
     projectId: "kerupuk-udang",
     storageBucket: "kerupuk-udang.appspot.com",
     messagingSenderId: "123456789",
     appId: "1:123456789:web:abcdef"
   };
   ```

   Salin nilai-nilai ini.

## 2. Isi konfigurasi di index.html

Buka `index.html`, cari bagian ini di dekat akhir file (bertanda komentar `KONFIGURASI FIREBASE`), lalu ganti dengan nilai dari langkah 1:

```js
const firebaseConfig = {
  apiKey: "GANTI_DENGAN_API_KEY",
  authDomain: "GANTI.firebaseapp.com",
  projectId: "GANTI_PROJECT_ID",
  storageBucket: "GANTI.appspot.com",
  messagingSenderId: "GANTI",
  appId: "GANTI"
};
```

Simpan file.

## 3. Deploy ke GitHub Pages

1. Buat repository baru di GitHub, misalnya `kerupuk-udang-app`.
2. Upload `index.html` (dan `README.md` ini kalau mau) ke repository tersebut — bisa lewat web GitHub (**Add file → Upload files**) atau lewat git:
   ```bash
   git init
   git add index.html
   git commit -m "Buku kalkulasi produksi kerupuk udang"
   git branch -M main
   git remote add origin https://github.com/USERNAME/kerupuk-udang-app.git
   git push -u origin main
   ```
3. Di repository GitHub → **Settings → Pages** → di bagian **Source**, pilih branch `main` dan folder `/ (root)` → **Save**.
4. Tunggu 1–2 menit, situs akan aktif di:
   `https://USERNAME.github.io/kerupuk-udang-app/`

## 4. Amankan Firestore (penting!)

Test mode Firestore terbuka untuk siapa saja selama 30 hari lalu terkunci otomatis. Untuk penggunaan jangka panjang, atur aturan akses di **Firestore Database → Rules**. Contoh aturan sederhana — publik boleh baca & tulis (cocok kalau aplikasi ini hanya untuk kamu sendiri dan linknya tidak disebar):

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /masters/{id} { allow read, write: if true; }
    match /produksi/{id} { allow read, write: if true; }
    match /penjualan/{id} { allow read, write: if true; }
  }
}
```

Kalau ingin lebih aman (hanya kamu yang bisa menyimpan/menghapus data), tambahkan Firebase Authentication dan ubah aturan menjadi `allow read, write: if request.auth != null;` — beri tahu saya kalau mau saya bantu tambahkan bagian login-nya.

## Cara pakai aplikasi

### 1. Master Resep
Buat satu master untuk tiap jenis kerupuk (misal "Kerupuk Puli", "Kerupuk Kotak"). Isi output standar (kg) dan daftar item resep beserta biayanya **untuk skala output standar itu**. Biaya per kg dihitung otomatis (total biaya resep ÷ output standar).

### 2. Produksi
Pilih master, isi tanggal, dan jumlah kg yang benar-benar diproduksi kali ini — boleh sama atau berbeda dari output standar. Aplikasi otomatis menghitung skala (misal 50% kalau produksi cuma setengah resep) dan menyesuaikan semua biaya bahan secara proporsional. Rincian biaya hasil skala ini masih bisa diedit manual sebelum disimpan (misal harga bahan naik). Setiap batch produksi punya stok awal = jumlah kg yang diproduksi.

### 3. Penjualan
Pilih batch produksi mana yang stoknya mau dijual (dropdown hanya menampilkan batch yang masih ada sisa stok), isi tanggal, jumlah kg terjual, dan harga jual per kg. Bisa dicatat bertahap — satu batch produksi boleh dijual ke beberapa pembeli di tanggal berbeda-beda, stok akan otomatis berkurang tiap kali disimpan. Kalau jumlah yang diinput melebihi sisa stok, sistem menolak menyimpan.

### 4. Pembukuan
Menampilkan ringkasan (total pendapatan, total biaya, laba bersih, total stok yang belum terjual) dan buku besar — semua transaksi produksi (pengeluaran) dan penjualan (pemasukan) digabung urut tanggal dengan saldo berjalan.

## Struktur data di Firestore

**`masters/{id}`** — satu dokumen per jenis kerupuk
```
name: string                 // "Kerupuk Puli"
outputStandarKg: number      // 1200
hargaJualDefault: number     // 16000
recipeItems: [{ name, unit, cost }, ...]   // biaya di skala standar
totalBiayaStandar: number
createdAt: timestamp
```

**`produksi/{id}`** — satu dokumen per batch produksi
```
masterId, masterName: string
tanggal: string (YYYY-MM-DD)
jumlahKg: number              // hasil produksi aktual, bisa beda dari standar
items: [{ name, unit, cost }, ...]   // biaya hasil skala, snapshot saat disimpan
totalBiaya: number
stokTerjual: number           // bertambah otomatis tiap ada penjualan dari batch ini
catatan: string
createdAt: timestamp
```

**`penjualan/{id}`** — satu dokumen per transaksi jual
```
produksiId, masterId, masterName: string
tanggalProduksi: string        // referensi tanggal batch asal
tanggal: string (YYYY-MM-DD)   // tanggal jual
jumlahKg: number
hargaPerKg: number
totalRp: number
pembeli: string
createdAt: timestamp
```

Stok dan penjualan disinkronkan lewat Firestore **transaction** (`runTransaction`), jadi aman dari race condition kalau ada dua penjualan disimpan hampir bersamaan.
