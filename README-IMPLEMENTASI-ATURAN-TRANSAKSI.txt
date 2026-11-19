IMPLEMENTASI ATURAN TRANSAKSI TOKO SIAP ANTAR
Tanggal: 19 November 2026

PERUBAHAN YANG DITERAPKAN PADA PAKET INI
1. Checkout saldo pelanggan dibuat atomik: debit saldo pelanggan, pembuatan seluruh order per mitra, dan catatan ewallet_transactions dilakukan dalam satu Firestore transaction. Jika salah satu write ditolak Rules, seluruh transaksi gagal; tidak ada order yang dibuat dengan status pembayaran palsu dan tidak ada fallback yang mengubah saldo_ewallet menjadi transfer.
2. Order saldo pelanggan menyimpan danaTertahan=true, danaTertahanNominal, dan danaTertahanStatus='tertahan'. Dana tersebut tidak dikreditkan ke saldo admin ketika checkout.
3. Persetujuan admin untuk transfer/QRIS meminta konfirmasi eksplisit bahwa pembayaran benar-benar telah diterima. Setelah itu order menyimpan danaTertahan=true sampai settlement.
4. Status selesai oleh mitra/kurir sekarang berarti menunggu verifikasi admin. Status akhir setelah admin memproses saldo adalah terverifikasi.
5. Penyelesaian COD ojek oleh kurir tidak lagi memotong saldo. Status hanya berubah menjadi selesai dan antrean verifikasi admin.
6. Settlement admin menghitung pembagian berdasarkan jenis order, COD/non-COD, dan apakah kurir ditugaskan. Saldo mitra/kurir/admin, catatan mutasi, dan status terverifikasi ditulis dalam satu transaksi Firestore.
7. COD kurir: saldo kurir diperiksa sebelum order diambil dan diperiksa ulang di dalam transaksi pengambilan. Pada verifikasi admin, kurir didebit 95% produk + fee admin; mitra dikredit 95% produk; admin dikredit 5% produk + 5% ongkir.
8. COD mitra sendiri: cash tetap di mitra; saat verifikasi admin saldo mitra didebit 5% produk + 5% ongkir dan admin dikredit fee tersebut.
9. COD ojek: saat verifikasi admin saldo kurir didebit 5% ongkir; tidak ada kredit saldo kurir.
10. Non-COD mitra sendiri: mitra dikredit 95% produk + 95% ongkir; admin 5% produk + 5% ongkir.
11. Non-COD dengan kurir: mitra dikredit 95% produk, kurir 95% ongkir, admin 5% produk + 5% ongkir.
12. Jalur settlement lama di halaman Semua Pesanan Admin diarahkan ke E-Wallet Admin. Tombol perbaikan saldo lama disembunyikan untuk mencegah pembagian ganda.
13. Konfirmasi penerimaan pelanggan tidak lagi mengubah saldo atau melakukan settlement. Refund hanya dilakukan jika dana Non-COD benar-benar tertahan/terverifikasi, dan refund dicatat dalam mutasi.
14. Status pembayaran tidak lagi ditimpa oleh status penyelesaian pengantaran di halaman mitra/kurir.

CONTOH PERHITUNGAN YANG DIUJI
Produk Rp100.000 + ongkir Rp10.000:
- Non-COD mitra sendiri: mitra Rp104.500, admin Rp5.500.
- Non-COD lewat kurir: mitra Rp95.000, kurir Rp9.500, admin Rp5.500.
- COD mitra sendiri: cash Rp110.000 di mitra, debit saldo mitra Rp5.500, admin menerima Rp5.500.
- COD lewat kurir: cash Rp110.000 di kurir, debit saldo kurir Rp100.500, mitra menerima saldo Rp95.000, admin Rp5.500, cash bersih kurir Rp9.500.
- Ojek ongkir Rp20.000: cash Rp20.000 di kurir, debit saldo kurir Rp1.000, cash bersih kurir Rp19.000.

BATASAN / LANGKAH WAJIB SEBELUM PRODUKSI
A. Paket ZIP sumber tidak menyertakan Firestore Security Rules atau Cloud Functions. Karena itu, perubahan ini adalah implementasi sisi-klien dan BELUM boleh dianggap aman untuk uang sungguhan.
B. Rules harus diperbarui untuk mengizinkan hanya role admin yang melakukan settlement dan hanya transisi status yang sah. Pelanggan tidak boleh bebas mengubah saldo atau membuat ledger kredit/debit sendiri.
C. Untuk keamanan produksi, debit checkout, settlement, refund, dan COD reserve harus dipindahkan ke Cloud Functions / backend tepercaya. Pemeriksaan role di HTML/JavaScript bukan batas keamanan.
D. Perubahan checkout memakai Firestore transaction multi-dokumen dan menulis ewallet_transactions bersama order. Jika Rules lama tidak mengizinkan field/collection tersebut, checkout saldo akan gagal seluruhnya secara aman. Jangan mengatasi dengan membuka Rules secara luas.
E. Persetujuan admin transfer/QRIS adalah penanda bahwa admin telah memeriksa uang benar-benar diterima. Sistem tidak dapat membuktikan transfer bank hanya dari flag di dokumen.
F. Tidak ada pengujian Firebase live, transaksi bersamaan, atau emulator Rules karena file Rules/proyek backend tidak disertakan. Uji di Firebase Emulator dan staging sebelum deploy.
G. Order lama dengan status selesai / adminFeeDibayar / saldoDibagi yang dibuat oleh logika lama perlu rekonsiliasi manual. Jangan jalankan settlement ulang pada data lama tanpa audit saldo dan mutasi.

VALIDASI STATIS
- Node.js syntax check pada 41 blok script dari file yang diubah: lulus.
- Seluruh paket: 129 blok script inline ditemukan; 128 lolos pemeriksaan regex/Node. Satu blok pada daftar-mitra-usaha.html berisi teks script tertanam yang membuat ekstraksi regex sederhana tidak valid; tidak diubah dalam pekerjaan ini.
- Tidak ada koneksi langsung ke Firebase yang diuji.
