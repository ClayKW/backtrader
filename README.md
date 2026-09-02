# Meta Prompt: Deploy Pipeline AI Strategy Generator + Backtester

Gunakan prompt ini sebagai instruksi awal untuk AI coding agent (Claude Code, Cursor, dll) yang akan membangun dan deploy sistem ini dari nol.

---

## Konteks proyek

Bangun sistem otomatis yang men-generate strategi trading dari data candle historis (CSV), memfilter strategi duplikat, mengubahnya jadi kode Python, mem-backtest dengan `backtrader`, dan menyimpan hasil yang berhasil ke database — semua berjalan otomatis dalam loop tanpa intervensi manual di setiap step, dengan auto-fix kalau kode yang digenerate error.

Instrumen: XAUUSD (gold spot). Data historis diexport dari MT5 History Center, format CSV dengan kolom OHLCV + indikator (Bollinger Bands, RSI, MACD, ATR, MA) yang sudah dihitung MT5 native.

## Arsitektur

Dua service terpisah, dideploy di Railway, masing-masing di-budget untuk **1GB RAM / 1 vCPU**:

**Service 1 — Generator** (node terpisah dari tester)
- Menerima window data candle (default 2000 candle) dari CSV mentah
- Panggil LLM API (Groq/OpenRouter, bukan model lokal — supaya CPU server tidak menanggung beban inferensi) untuk generate strategi dalam **format SOP terstandarisasi**:
  ```
  Entry (AND — semua kondisi harus true bersamaan):
  a. <kondisi 1>
  b. <kondisi 2>
  c. <kondisi N, boleh lebih>
  TP: <nilai>
  SL: <nilai>
  ```
- Sebelum menyimpan/lanjut, jalankan **dedupe check**: normalisasi kondisi (urutkan alfabetis berdasarkan nama indikator, supaya kombinasi kondisi yang urutannya beda tapi isinya sama tetap dianggap identik), lalu hash struktur (indikator + operator + threshold + TP/SL) dan bandingkan ke tabel `generated_strategies` (signature saja, bukan hasil performa — tabel ini sengaja dipisah dari tabel hasil).
- Kalau signature sudah ada → generate ulang (loop), dengan prompt LLM disuntik **daftar ringkas** signature yang sudah pernah dibuat (1 baris per strategi, bukan detail lengkap) supaya LLM sadar dan menghindari pola serupa. Jangan kirim seluruh histori penuh ke context — bisa membengkak dan mahal.
- Kalau signature unik → simpan signature ke `generated_strategies`, lanjut minta LLM convert SOP itu jadi kode strategi `backtrader` (class `bt.Strategy` dengan kondisi AND eksplisit, TP/SL sesuai SOP).
- Kirim job (kode Python yang di-generate) ke Service 2 lewat tabel job queue sederhana (lihat bagian Job Queue di bawah) — tidak perlu message broker (RabbitMQ/Redis), tabel database dengan kolom status sudah cukup di skala ini.

**Service 2 — Tester** (mesin terpisah, jalankan backtest)
- Poll tabel job untuk job berstatus `pending`, ambil satu, ubah jadi `processing`
- Jalankan kode strategi di `backtrader` terhadap CSV data (pisahkan window data generate vs window backtest supaya tidak overfitting — jangan backtest di candle yang sama persis dipakai untuk generate strategi)
- Validasi hanya sebatas **script jalan tanpa error / error** — tidak ada gate kualitas performa di level ini
- Kalau **error**: tulis log error lengkap (stack trace) ke kolom job, set status `error`, kirim balik ke Service 1 (via job status) supaya LLM diminta perbaiki kode berdasarkan log tsb. Retry dengan **limit** (default 5x per strategi) supaya tidak infinite loop — walau API LLM unlimited, tetap batasi supaya job yang genuinely broken tidak nyangkut selamanya (flag `max_retries_exceeded` kalau limit tercapai, jangan looping terus).
- Kalau **berhasil jalan**: tarik semua metrik dari `cerebro.run()` analyzers (profit factor, win rate, max drawdown, jumlah trade, Sharpe ratio, equity curve) dan simpan **lengkap** ke tabel `backtest_results` (tabel terpisah dari `generated_strategies`) — supaya nanti proses filter/sortir strategi bagus tidak perlu re-run backtest dari awal.

## Skema database (PostgreSQL)

```sql
CREATE TABLE generated_strategies (
    id SERIAL PRIMARY KEY,
    signature_hash TEXT UNIQUE NOT NULL,   -- hash dari struktur SOP yang sudah dinormalisasi
    sop_text TEXT NOT NULL,                -- format SOP asli (entry a/b/c, TP, SL)
    created_at TIMESTAMP DEFAULT now()
);

CREATE TABLE strategy_jobs (
    id SERIAL PRIMARY KEY,
    strategy_id INT REFERENCES generated_strategies(id),
    python_code TEXT NOT NULL,
    status TEXT DEFAULT 'pending',         -- pending | processing | error | done | max_retries_exceeded
    retry_count INT DEFAULT 0,
    last_error_log TEXT,
    updated_at TIMESTAMP DEFAULT now()
);

CREATE TABLE backtest_results (
    id SERIAL PRIMARY KEY,
    strategy_id INT REFERENCES generated_strategies(id),
    profit_factor NUMERIC,
    win_rate NUMERIC,
    max_drawdown NUMERIC,
    total_trades INT,
    sharpe_ratio NUMERIC,
    equity_curve JSONB,
    tested_at TIMESTAMP DEFAULT now()
);
```

## Requirement teknis

- Python 3.11+, `backtrader`, `pandas`, `psycopg2` / `sqlalchemy`, `fastapi` + `uvicorn` (kalau butuh endpoint trigger manual), `requests` (panggil LLM API eksternal)
- Retry limit untuk auto-fix loop: konfigurasi via env var `MAX_RETRY_PER_STRATEGY` (default 5)
- Ukuran window generate: env var `CANDLE_WINDOW_SIZE` (default 2000)
- Pisahkan `.env` untuk kredensial LLM API dan koneksi PostgreSQL antar kedua service
- Nonaktifkan `cerebro.plot()` di server (headless, tanpa GUI) — ambil hasil dari `analyzers` saja, jangan render chart
- Karena resource 1GB/1vCPU per service: hindari load semua data candle sekaligus kalau timeframe M1 dan rentang tahun panjang — proses per-batch/tahun kalau perlu (lihat catatan RAM di bagian bawah)
- Deploy tiap service sebagai container terpisah di Railway, environment variable untuk koneksi antar service (URL job queue / shared PostgreSQL instance)

## Yang perlu diminta ke coding agent secara eksplisit

1. Setup struktur project 2 folder terpisah: `generator/` dan `tester/`, masing-masing punya `Dockerfile` dan `requirements.txt` sendiri untuk deploy independen di Railway
2. Implementasi fungsi normalisasi + hashing signature strategi (sort kondisi, canonical format sebelum hash)
3. Implementasi prompt template untuk LLM: satu untuk generate SOP dari data candle, satu untuk convert SOP → kode backtrader, satu untuk perbaiki kode berdasarkan error log
4. Implementasi polling loop di Service 2 (cek job `pending` tiap interval tertentu, misal tiap 10-30 detik)
5. Implementasi retry counter dan guard `max_retries_exceeded`
6. Migrasi/schema PostgreSQL sesuai skema di atas
7. Script deploy ke Railway (railway.json / railway.toml per service)

---

Copy seluruh isi di atas sebagai instruksi awal ke coding agent, lalu lampirkan sample CSV data candle (dengan kolom OHLCV + indikator) sebagai referensi format input.
