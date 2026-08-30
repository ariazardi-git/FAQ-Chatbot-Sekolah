# FAQ-Chatbot-Sekolah

# FAQ Chatbot Sekolah — Politeknik Caltex Riau (PCR)

Chatbot FAQ berbasis n8n yang menjawab pertanyaan umum calon mahasiswa/masyarakat tentang Politeknik Caltex Riau, dengan kemampuan klasifikasi intent otomatis, routing ke jawaban spesifik per kategori, dan fallback ke AI Agent umum (dilengkapi Calculator Tool) untuk pertanyaan di luar kategori utama.

## Apa yang Dilakukan Workflow Ini

Workflow ini menerima pertanyaan dari user lewat jendela chat n8n, lalu:

1. **Mengklasifikasikan** pertanyaan ke salah satu dari 4 kategori: `pendaftaran`, `biaya`, `jurusan`, atau kategori lain (ditangani lewat fallback).
2. **Merutekan** pertanyaan ke jawaban yang sesuai berdasarkan hasil klasifikasi.
3. Untuk pertanyaan di luar 3 kategori spesifik, diteruskan ke **AI Agent-FAQ** yang punya akses ke 10 Q&A umum tentang PCR dan **Calculator Tool** untuk pertanyaan yang butuh perhitungan.
4. Semua hasil digabung kembali dan ditampilkan sebagai satu balasan ke user.

## Struktur Workflow (Node)

| Node | Fungsi |
|---|---|
| `When chat message received` | Trigger — menerima pesan dari user lewat jendela chat |
| `AI Agent - Classifier` | Mengklasifikasikan pertanyaan user ke salah satu dari 4 kategori, output dalam format JSON terstruktur (`category`, `confidence`, `reasoning`) |
| `Structured Output Parser` | Memaksa output AI Agent - Classifier selalu dalam format JSON yang konsisten |
| `Switch` | Merutekan berdasarkan `category` hasil klasifikasi ke branch yang sesuai. Kategori yang tidak cocok rule manapun otomatis masuk ke **Fallback Output** |
| `Pendaftaran` (Set) | Jawaban statis untuk pertanyaan seputar pendaftaran mahasiswa baru |
| `Biaya` (Set) | Jawaban statis untuk pertanyaan seputar biaya kuliah |
| `Jurusan` (Set) | Jawaban statis untuk pertanyaan seputar program studi/jurusan |
| `AI Agent-FAQ` | Menjawab pertanyaan di luar 3 kategori spesifik menggunakan 10 Q&A umum tentang PCR, dilengkapi **Calculator Tool** untuk pertanyaan berhitung |
| `Calculator` | Tool tambahan untuk AI Agent-FAQ, dipakai saat user bertanya sesuatu yang butuh perhitungan matematis |
| `Merge` | Menggabungkan hasil dari semua branch menjadi satu output final yang ditampilkan ke user |

## Cara Pakai

1. Buka workflow di n8n, klik **"Open chat"** di bagian bawah editor.
2. Ketik pertanyaan seputar PCR (pendaftaran, biaya, jurusan) atau pertanyaan umum lainnya.
3. Chatbot akan otomatis mengklasifikasikan dan menjawab sesuai kategori.

**Catatan penting:** Field `output` pada setiap node `Set` (Pendaftaran, Biaya, Jurusan) WAJIB bernama persis `output` (huruf kecil) agar jendela chat n8n dapat membaca dan menampilkan balasannya.

## Test Cases

| # | Pertanyaan | Kategori yang Diharapkan | Hasil |
|---|---|---|---|
| 1 | "Bagaimana cara mendaftar jadi mahasiswa baru?" | Pendaftaran | ✅ Berhasil — diarahkan ke node Pendaftaran, jawaban berisi link pmb.pcr.ac.id/panduan |
| 2 | "Berapa biaya kuliahnya?" | Biaya | ✅ Berhasil — diarahkan ke node Biaya, jawaban berisi link pmb.pcr.ac.id/panduan/biaya |
| 3 | "Jurusan apa saja yang ada di PCR?" | Jurusan | ✅ Berhasil — diarahkan ke node Jurusan, jawaban berisi link pmb.pcr.ac.id/prodi |
| 4 | "Siapa rektor PCR?" | Fallback → AI Agent-FAQ | ✅ Berhasil — dijawab sopan bahwa info tidak tersedia, disertai saran hubungi PMB PCR |
| 5 | "8474 x 391" / "11 x 11" | Fallback → AI Agent-FAQ (pakai Calculator Tool) | ✅ Berhasil — Calculator Tool terpanggil (terlihat di Logs), hasil hitungan akurat |

## Catatan Teknis & Troubleshooting

- **Structured Output Parser** wajib pakai contoh JSON berupa **satu object** dengan schema konsisten (bukan array berisi object dengan field berbeda-beda), contoh: `{ "category": "pendaftaran", "confidence": "high", "reasoning": "..." }`.
- **Switch Fallback Output** dipakai (bukan rule string-matching manual untuk kategori ke-4) karena AI classifier tidak selalu menghasilkan kata yang persis sama untuk kategori "lainnya" (misal kadang muncul `"others"` alih-alih `"lainnya"`). Fallback Output menangkap semua item yang tidak cocok 3 rule spesifik, sehingga lebih tahan terhadap variasi output AI.
- **AI Agent-FAQ** menggunakan expression `{{ $('When chat message received').first().json.chatInput }}` (bukan `.item`) untuk mengambil pertanyaan asli user, karena data yang lewat jalur Fallback pada Switch node tidak selalu membawa referensi *paired item* yang utuh ke node trigger.
- Jika muncul error `OpenAI: Rate limit reached` atau sejenisnya, ini murni soal kuota API dari provider model (OpenRouter/OpenAI), bukan masalah konfigurasi workflow.

## Kredensial yang Dibutuhkan

- 1x credential OpenRouter (atau provider LLM lain) untuk Chat Model di `AI Agent - Classifier`
- 1x credential OpenRouter (atau provider LLM lain) untuk Chat Model di `AI Agent-FAQ`
- Tidak perlu credential tambahan untuk Calculator Tool
