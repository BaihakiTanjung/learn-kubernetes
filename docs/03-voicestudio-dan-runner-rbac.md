# VoiceStudio, RBAC Runner & Perbaikan Penjadwalan

Status: **Deployment VoiceStudio + SA/RBAC runner + nodeSelector template + pin
office-3d SELESAI 26 November 2026 ~01:43–02:09; lisensi diterima + voice
diimpor + DNS bypass + STT + profil performa SELESAI 26 Nov ~09:36–11:46;
hasil GGUF SELESAI 26 Nov (§1.10) — node VPS tidak bisa menjalankan engine
TTS Indonesia apa pun (Xeon E5620 tanpa AVX, RAM 7,75 GB); uji Laras TTS
SELESAI 26 Nov (§1.11) — biner mati SIGILL, AVX2 juga menutup jalur ini;
rebuild Laras di VPS GAGAL 26 Nov (§1.12) — pivot: Laras JALAN di sandbox
Muse, render tetap di VPS; draf office-3d SELESAI 26 Nov (§1.13);
Deployment render standby SELESAI 26 Nov 22:12 (§1.14) — smoke test
still lulus, rute baked video-audio ditutup untuk T-013.**
Yang MENUNGGU tangan/keputusan pemilik server dipisah tegas di
§3 — jangan dicampur dengan yang sudah selesai. Tidak ada perubahan di klaster
pada penulisan dokumen ini; semua yang di bawah adalah yang benar-benar
terjadi dan terverifikasi pada jam tersebut.

## 1. Yang SUDAH selesai

### 1.1 VoiceStudio jalan di vps-38 (namespace `render`)

Keputusan yang mendasarinya: VoiceStudio = Deployment persistent (replicas 1) yang
tetap hangat — cold start berarti pull image + load model 3,2 GB — sedangkan
render = satu Job per video yang dihapus setelah selesai (§2).

| Resource | Nilai |
| --- | --- |
| PVC | `voicestudio-data` 10Gi, storage class `local-path` |
| Deployment | `voicestudio`, replicas 1, `nodeSelector: { workload: render }` |
| Image | `ghcr.io/debpalash/voicestudio:stable` (debpalash/VoiceStudio, AGPL-3.0) |
| Ukuran image terukur | **~5,6 GiB**; pull pertama **18 mnt 23 dtk** |
| Service | `voicestudio`, ClusterIP, port **3900** |
| Env | `OMNIVOICE_API_KEY` dari Secret `voicestudio-api-key` (`secretKeyRef`); `OMNIVOICE_DOTS_TTS_PRECISION=bfloat16` (§1.10) |
| Pod | 1/1 Running di **vps-38** |
| Limit memori | 7 GiB sejak §1.10 (dari 4 GiB; peak generate OmniVoice terukur 3,4 GiB) |
| DNS pod | `dnsPolicy: None` + nameserver 8.8.8.8 / 1.1.1.1 (§1.6) |
| STT | faster-whisper `base` terinstal (§1.7) |
| Profil TTS | `tier=fast` (§1.8) |

Ringkasan manifest (nilai sensitif hanya sebagai placeholder — lihat §3.3):

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: voicestudio-data
  namespace: render
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: local-path
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: voicestudio
  namespace: render
spec:
  replicas: 1
  selector:
    matchLabels: {app: voicestudio}
  template:
    metadata:
      labels: {app: voicestudio}
    spec:
      nodeSelector:
        workload: render
      dnsPolicy: None
      dnsConfig:
        nameservers:
        - 8.8.8.8
        - 1.1.1.1
      containers:
      - name: voicestudio
        image: ghcr.io/debpalash/voicestudio:stable
        ports:
        - containerPort: 3900
        env:
        - name: OMNIVOICE_API_KEY
          valueFrom:
            secretKeyRef:
              name: voicestudio-api-key
              key: api-key
        - name: OMNIVOICE_DOTS_TTS_PRECISION
          value: bfloat16
        resources:
          limits:
            memory: 7Gi
---
apiVersion: v1
kind: Service
metadata:
  name: voicestudio
  namespace: render
spec:
  selector: {app: voicestudio}
  ports:
  - port: 3900
```

Verifikasi (26 Nov ~02:09):

```text
GET /health → 200 {"status":"ok","device":"cpu","engine":"omnivoice","version":"0.5.8"}
GET /v1/audio/voices → 200 (14 alias OpenAI + demo0001)
```

Verifikasi (26 Nov ~11:46, setelah DNS bypass §1.6):

```text
getent hosts huggingface.co (dari dalam pod) → exit 0
curl https://huggingface.co → HTTP/2 200
GET /health → 200 (pod baru, setelah restart)
weights ~3,1 GB terunduh ke HF_HOME di PVC
```

**Koreksi angka:** image VoiceStudio terukur **~5,6 GiB**, bukan 17 GB seperti
perkiraan di [docs/02-garbage-collection-optimasi.md](02-garbage-collection-optimasi.md).
Angka 17 GB itu perkiraan sebelum image ter-pull dan **tidak lagi dipakai** —
termasuk untuk hitungan prune di docs/02 §3A (image VoiceStudio tetap ikut
terhapus bila di-prune, karena Deployment-nya tidak selalu punya pod yang jalan,
tapi pull ulangnya ~18 menit, bukan "10+ menit untuk 17 GB").

**Status generate suara (26 Nov ~11:46):** lisensi sudah diterima (§3.1) dan
4 voice sudah diimpor (§3.2), weights sudah terunduh — tapi **test generate
gagal**: `POST /v1/audio/speech` (voice gufron) → HTTP 503 tepat di 600,28 dtk
(§1.9). TTS OmniVoice di CPU VPS **tidak layak produksi**; narasi tetap
mengandalkan VoiceStudio laptop sampai ada keputusan mesin baru (§3.6).
Percobaan engine ringan GGUF (§1.10) juga gagal di hardware ini — vonisnya
di §1.10 — dan uji Laras TTS (§1.11) juga gagal: biner mati SIGILL karena
AVX2. Rebuild Laras dari source di VPS pun gagal (§1.12) — tapi Laras
**jalan di sandbox Muse** (§1.12), dan itulah jalur narasi saat ini.
Opsi lanjut di §3.6.

### 1.2 ServiceAccount + RBAC minimal untuk GitHub runner

Runner GitHub (repo `birunix-render`) mengakses klaster **hanya** lewat
ServiceAccount ini — tanpa kubeconfig admin, tanpa akses namespace lain.

| Resource | Isi |
| --- | --- |
| ServiceAccount | `github-render-runner` (ns `render`) |
| Role | `github-render-runner` (ns `render`): `create/get/list/watch` pada `jobs` (batch) + `pods` (core) — **hanya di ns `render`** |
| RoleBinding | `github-render-runner` → SA di atas |
| Secret | `github-render-runner-token`, type `kubernetes.io/service-account-token` |

Aturan yang dijaga: Role tidak memberi akses Secret, Deployment, ConfigMap, atau
namespace lain. Kalau runner butuh operasi baru, tambahkan verb/resource ke Role
ini secara eksplisit — jangan ganti dengan ClusterRole.

**Membaca token (dilakukan pemilik server sendiri, di vps-29)** — hasilnya
disimpan sebagai GitHub secret `K8S_RENDER_TOKEN` di repo `birunix-render`
(§3.4). Nilai token **tidak pernah ditempel di chat dan tidak pernah masuk repo**:

```bash
sudo /usr/local/bin/kubectl get secret github-render-runner-token -n render \
  -o jsonpath='{.data.token}' | base64 -d
```

### 1.3 `nodeSelector: workload=render` di 3 template Job render

Ditambahkan ke pod template (file, bukan objek klaster — Job dibuat dari template
ini saat render dijalankan):

- `remotion-clip-job.yaml`
- `render-full-job.yaml`
- `render-smoke-job.yaml`

```yaml
      nodeSelector:
        workload: render
```

Efek: Job render (peak ~1,6 GiB) lari ke vps-38 yang memorinya longgar,
bersamaan dengan VoiceStudio (peak 3,4 GiB) masih muat — 3,4 + 1,6 = 5,0 GiB
dari ~7,75 GiB allocatable vps-38. Aturan yang dipertahankan: **satu render
dalam satu waktu**. Ini menutup docs/02 §4A ("Pindahkan Job render ke vps-38").

### 1.4 Deployment office-3d di-pin ke vps-29

Deployment `muse-hq-office-3d` (ns `apps`) sempat rollout dan pod barunya
tersangkut di vps-38: `Pending` → `ContainerCreating` selama 18 menit dengan
`FailedMount` — volume `hostPath` `/home/rlordz12/hq-office-3d` **hanya ada
di vps-29**. Pod lama tetap Running di vps-29, jadi situs tidak sempat down.

Perbaikan (disetujui pemilik server, 26 Nov ~01:43):

```bash
sudo -n /usr/local/bin/kubectl patch deploy muse-hq-office-3d -n apps \
  --type=strategic \
  -p '{"spec":{"template":{"spec":{"nodeSelector":{"kubernetes.io/hostname":"vps-29"}}}}}'
```

Hasil: rollout sukses, pod baru 1/1 Running di vps-29, pod lama Terminating,
pod nyangkut di vps-38 hilang, situs `https://office3d.103.77.78.114.sslip.io/`
→ **HTTP 200**.

**Pelajaran — hostPath harus di-pin ke node pemilik path.** Tanpa
`nodeSelector`, scheduler bebas menempatkan pod di node mana pun, dan pod yang
butuh path lokal node tertentu akan gagal mount di node lain. Aturan umum:
hostPath → pin `kubernetes.io/hostname`; kalau workload harus bisa jalan di
node mana pun, pakai PVC (atau storage bersama), bukan hostPath.

### 1.5 Probe port 6443 dari internet

Diuji dari luar klaster apakah GitHub runner bisa mencapai API server lewat
IP publik `https://103.77.78.114:6443`:

- TCP terjangkau, **TLS handshake OK** (SAN sertifikat memuat 103.77.78.114).
- Tapi server **menutup koneksi tanpa respons HTTP**: `curl 52` (Empty reply
  from server), `/healthz` → `000`.

Kesimpulan: **rute runner → API via IP publik BELUM terbukti**. `--insecure-skip-tls-verify`
bukan solusi — TLS-nya sudah berhasil, yang gagal adalah respons HTTP-nya.
Diagnosa lanjut + keputusan rute ada di §3.5.

### 1.6 Opsi B — DNS bypass pod VoiceStudio (diterapkan 26 Nov ~10:51–11:46)

**Latar — data plane jaringan pod vps-38 rusak** (diagnosis Zenith, 26 Nov
10:12): pod di vps-38 tidak bisa mencapai IP pod mana pun, satu node maupun
lintas node — akibatnya DNS via kube-dns (`10.43.0.10`) gagal total
(`getent` menggantung, `httpx.ConnectError: Temporary failure in name
resolution`). Yang tetap jalan: pod→gateway, pod→IP node, dan pod→internet
langsung, termasuk DNS mentah UDP ke 8.8.8.8 / 1.1.1.1. Restart k3s-agent di
vps-38 (10:38) **tidak memperbaiki** — pod baru pun gagal dengan pola
identik; flannel start bersih pasca-restart, jadi hipotesis urutan init
iptables/flannel melemah. Hipotesis berikutnya (belum terverifikasi): aturan
iptables FORWARD / kube-router vps-38 yang spesifik-destinasi; bridge/VXLAN
FDB stale; `modprobe nft-expr-counter` gagal. Catatan: **render Job di
vps-38 yang butuh DNS kena masalah yang sama.**

Keputusan pemilik server 26 Nov 10:51: **Opsi B — bypass kube-dns khusus pod
VoiceStudio** (tercepat; Opsi A perbaiki akar sudah dicoba lewat restart
agent dan gagal; Opsi C pre-seed weights tidak dipilih).

Perubahan pada Deployment `voicestudio` (ns `render`):

```yaml
spec:
  template:
    spec:
      dnsPolicy: None
      dnsConfig:
        nameservers:
        - 8.8.8.8
        - 1.1.1.1
```

**Prosedur penerapan — scale-to-0 → patch → scale-to-1, bukan rollout
restart biasa.** Alasannya insiden ~10:41: sebuah rollout sempat deadlock
`CrashLoopBackOff` dengan `DataDirectoryBusy` — pod lama masih memegang lease
`/app/omnivoice_data` sementara pod baru butuh direktori yang sama. Men-scale
ReplicaSet lama ke 0 dulu memastikan tidak ada dua pod yang berebut lease.
Manifest lokal `voicestudio-vps38.yaml` diperbarui agar sama dengan live
state (jangan sampai apply berikutnya mengembalikan setting lama).

**Konsekuensi yang diharapkan, bukan kegagalan:** dengan `dnsPolicy: None`,
pod **tidak lagi resolve nama `*.svc.cluster.local`** — `getent
kubernetes.default.svc.cluster.local` gagal adalah perilaku yang benar.
VoiceStudio tidak membutuhkannya (hanya butuh huggingface.co + API
eksternal). Service tetap merutekan trafik ke pod seperti biasa (itu urusan
kube-proxy di node, bukan DNS pod).

Verifikasi (26 Nov ~11:46): `getent hosts huggingface.co` → exit 0;
`curl https://huggingface.co` → HTTP/2 200; weights **~3,1 GB terunduh** ke
`HF_HOME=/app/omnivoice_data/huggingface` di PVC (cache bertahan restart
pod).

**Batasan:** ini workaround per-pod, **bukan perbaikan jaringan vps-38** —
pod-to-pod tetap rusak. Perbaikan akar masih menunggu (§3.6).

### 1.7 STT faster-whisper-base terinstal di VoiceStudio VPS

Model STT `faster-whisper-base` terinstal di VoiceStudio VPS (26 Nov ~11:46):

| Item | Nilai |
| --- | --- |
| Model | faster-whisper `base` |
| Ukuran | 147.886.409 B (~141 MiB), 6 file |
| Status | `installed: true` |

Transkripsi untuk passage-selection tercatat berjalan di log pod — STT di
VPS sudah bisa dipakai (misalnya untuk memilih passage referensi voice),
terpisah dari masalah TTS di §1.9.

### 1.8 Profil performa `tier=fast` untuk TTS

Diatur 26 Nov ~11:46 via API VoiceStudio:

```bash
PUT /api/settings/performance-profile {"tier":"fast","family":"tts"}
```

Hasil: `effective.tts: fast` — diffusion steps **8** (dari 16),
postprocess **false**; profil global tetap `balanced` (hanya keluarga TTS
yang diubah).

**Bertahan setelah restart pod:** setting tersimpan sebagai
`overrides: {tts: "fast"}`, jadi pod baru tetap memakai tier fast tanpa
setel ulang. Catatan: tier fast mempercepat generate, tapi **tidak mengubah
kesimpulan §1.9** — bottleneck-nya CPU, bukan jumlah langkah.

### 1.9 Hasil test generate — TTS OmniVoice CPU VPS tidak layak produksi

Test generate pertama setelah DNS bypass (26 Nov ~11:46, voice gufron,
kalimat ~4 detik):

```text
POST /v1/audio/speech → HTTP 503 tepat di 600,28 dtk
```

- **Budget eksekusi server 600 dtk terlampaui:** cold load model ~3,5 mnt +
  generasi ~11 mnt untuk kalimat ~4 dtk (**~165x real-time**).
- CPU terpakai hanya **~3,4 core dari limit 7** — engine tidak memanfaatkan
  semua core yang tersedia.
- **3 voice lain tidak dicoba** — bottleneck yang sama berlaku untuk semuanya.
- `voice-tests/` tetap kosong: **tidak ada WAV yang dihasilkan**.
- Job yatim sempat wedged lalu dibereskan via restart pod; sistem bersih.

**Kesimpulan: OmniVoice di CPU VPS tidak layak produksi.** Opsi lanjut
menunggu keputusan pemilik server (§3.6); sampai ada keputusan, narasi
tetap lewat VoiceStudio laptop (rute terbukti jalan).

### 1.10 Hasil GGUF — node VPS tidak bisa menjalankan engine TTS Indonesia apa pun

Keputusan pemilik 26 Nov 11:51: coba engine ringan GGUF di VPS (setelah
§1.9). Zenith menyelidiki katalog, menaikkan limit memori, memasang dua
engine Python, lalu menguji keduanya. Hasilnya: **bukan masalah engine —
masalah hardware node**.

#### Limit memori pod 4 GiB → 7 GiB

```bash
sudo /usr/local/bin/kubectl set resources deploy/voicestudio -n render \
  --limits=memory=7000Mi
```

Manifest lokal `voicestudio-vps38.yaml` diperbarui agar sama dengan live state
(jangan sampai apply berikutnya mengembalikan limit lama). Env
`OMNIVOICE_DOTS_TTS_PRECISION=bfloat16` juga ditambahkan ke manifest (§1.1).

#### Dua engine terinstal + import OK — lalu OOM (exit 137)

| Engine | Detail | Weights di PVC |
| --- | --- | --- |
| dots-tts (Python, 2B, 24 bahasa incl. id, Apache-2.0) | sidecar venv, import OK | snapshot 4,9 GB (15 file) di HF cache PVC |
| VoxCPM2 (2B, 30 bahasa incl. id, Apache-2.0) | import OK | ~5 GB di PVC |

Keduanya mati OOM saat load model (exit 137), meski limit sudah 7 GiB:

- **dots-tts:** bobot BF16 2,198 miliar parameter; load default CPU memakai
  float32 → butuh **±9 GB RAM**. `bfloat16` di CPU **ditolak eksplisit oleh
  upstream** (`_check_torch_env` → `RuntimeError: Half-precision is not
  reliable on CPU`). Env `OMNIVOICE_DOTS_TTS_PRECISION=bfloat16` sudah sampai
  ke proses backend, tapi load fp32 sudah OOM duluan.
- **VoxCPM2:** LLM 2B bf16 ±4 GB + AudioVAE + ZipEnhancer/denoiser + torch +
  backend ±1,3 GB tetap melewati 7000Mi.
- Node vps-38 hanya **7,75 GB total** — tidak ada ruang untuk keduanya.

#### Penyebab GGUF mati: CPU Xeon E5620 (2010) tanpa AVX/AVX2/FMA

Terverifikasi dari `/proc/cpuinfo` di vps-29 dan di dalam pod vps-38: kedua
node pakai **Intel Xeon E5620 (2010) TANPA AVX/AVX2/FMA**. Akibatnya:

- Biner prebuilt **audio.cpp / dots-tts GGUF menolak jalan**:
  `CPU ISA level is lower than required`.
- **OmniVoice GGUF** butuh **GLIBC_2.43**, image pod glibc **2.39** → tidak
  bisa dieksekusi. Biner + lib sudah disalin ke PVC
  `/app/omnivoice_data/gguf-bin/bin/` (74 MB) untuk masa depan (node/image
  yang memenuhi syarat).
- **CosyVoice3 GGUF** butuh build dari source — masalah ISA yang sama.

Catatan: dari tiga kandidat GGUF awal, hanya dots-tts-soar yang resmi
mendukung Bahasa Indonesia (Breeze-TTS-2 hanya Inggris+Mandarin,
Fun-CosyVoice3 hanya 9 bahasa tanpa id) — tapi itu sudah tidak relevan di
node ini: **semua** jalur GGUF mati di ISA, bukan di dukungan bahasa.

#### Vonis hardware

**Node VPS ini tidak bisa menjalankan engine TTS berbahasa Indonesia apa pun
di VoiceStudio 0.5.8.** OmniVoice PyTorch satu-satunya yang jalan — dan itu
pun 165x real-time + budget 600 dtk, tidak layak produksi (§1.9). Opsi
lanjut ada di §3.6.

**Peringatan operasional:** dengan limit 7 GiB, VoiceStudio sendiri hampir
memenuhi node 7,75 GB — jangan jalankan render Job bersamaan dengan
load/generate VoiceStudio (7 + 1,6 GiB peak render > node; lihat §1.3).

### 1.11 Hasil uji Laras TTS — biner mati SIGILL (AVX2) di node ini

Keputusan pemilik 26 Nov 13:21: coba Laras TTS — mesin TTS Indonesian-first,
lokal, gratis — setelah riset Rhea 13:18. Laras (L.A.R.A.S = Lightweight Audio
Recognition & Artificial Speech, oleh MeowLabs, <https://laras.meowlabs.id>)
adalah lapisan produksi di atas Kyutai Pocket TTS + bobot
`anak10thn/pocket-tts-indonesian`: normalizer angka Indonesia, chunking per
kalimat, kloning dari referensi 10–15 detik, server Rust + ONNX int8 satu
biner. Lisensi kode MIT OR Apache-2.0; bobot CC-BY-4.0 (komersial diizinkan
dengan atribusi); **tidak ada gerbang lisensi** seperti VoiceStudio.

Uji langkah 1 — gate hardware (Zenith, 26 Nov 13:21–13:33): Job
`laras-test-gufron` di ns `render`, `nodeSelector: { workload: render }`
(vps-38), `dnsPolicy: None` + 8.8.8.8 / 1.1.1.1 (pola §1.6), initContainer
mengunduh 9 berkas model ±160 MB dari Hugging Face dengan verifikasi sha256,
PVC `laras-data` 3Gi, satu sintesis `takes=1` voice gufron, kalimat "Halo, ini
tes suara dari VoiceStudio di VPS."

| Tahap | Hasil |
| --- | --- |
| Pull image `ghcr.io/urdemonlord/laras-tts:0.2.1` | OK |
| Unduh model ±160 MB (9 berkas, sha256 terverifikasi) | OK, 21 detik |
| 4 ref MP3 masuk pod (gufron, anak-anak, resah, my voice) | OK |
| Start server Laras | **GAGAL — biner mati SIGILL, exit 132, `server.log` 0 byte** |

Model yang terunduh (semua sha256 cocok, total 164.385.066 byte):

```text
flow_lm_main_int8.onnx    75.865.936
mimi_encoder.onnx         39.259.746
mimi_decoder_int8.onnx    22.616.918
text_conditioner.onnx     16.388.310
flow_lm_flow_int8.onnx     9.935.594
tokenizer.model              311.347
bos_before_voice.npy           4.224
VALIDATION.txt                 2.494
bundle.json                      497
```

Penyebab: ONNX Runtime prebuilt di image Laras dikompilasi dengan AVX2;
CPU Xeon E5620 hanya SSE4.2 (§1.10). Polanya sama dengan biner GGUF
audio.cpp — bedanya, Laras tidak sempat mencetak apa pun (mati di instruksi
pertama), sementara GGUF setidaknya menolak dengan pesan "CPU ISA level is
lower than required".

**Vonis: Laras tidak jalan di node ini.** Langkah 2 (uji kualitas:
pengucapan gugus konsonan, kemiripan clone, perbedaan antar-voice) **tidak
dilakukan** — tidak ada WAV yang dihasilkan.

**Temuan infrastruktur — PVC `render-work` tidak bisa dipakai Job di
vps-38.** PV-nya terikat node vps-29 oleh nodeAffinity local-path (PVC itu
pertama dipakai di vps-29). Dibuat PVC baru `laras-data` 3Gi
(`WaitForFirstConsumer` → PV ikut dibuat di vps-38), berisi model Laras + 4
ref MP3, **masih ada**. Aturan: PVC local-path mengikat ke node konsumen
pertamanya — Job di vps-38 butuh PVC yang belum terikat ke vps-29.

**Bug driver:** `kubectl cp` gagal bila direktori tujuan belum ada di pod
(`tar: /data/ref: Cannot open: No such file or directory`) — `mkdir` dulu di
dalam pod sebelum `cp`.

Keadaan akhir: Job `laras-test-gufron` dihapus; Deployment VoiceStudio tidak
disentuh (1/1 Ready); tidak ada lisensi yang diterima (Laras tidak punya
gerbang lisensi). Opsi lanjut ada di §3.6. Manifest lokal Job + PVC +
driver ada di workspace (`vps-render/laras-test-gufron-job.yaml`,
`laras-data-pvc.yaml`, `laras-run-step1.sh`).

### 1.12 Rebuild Laras di VPS gagal — Laras JALAN di sandbox Muse (pivot 26 Nov 18:49)

Lanjutan §1.11. Opsi rebuild dari source (§3.6 butir 4) diuji — dan gugur.
Keputusan pemilik 26 Nov 18:49: **Laras dipindah ke sandbox Muse; render
tetap di VPS.**

#### Rebuild dari source di VPS — GAGAL

Job `laras-build` (ns `render`, vps-38; build ONNX Runtime dari source
dengan `target-cpu=x86-64`, mengikuti rencana §1.11) **Failed setelah
149 menit** dengan error:

```text
could not find native static library 're2'
```

— gagal saat `cargo build` paket `ort-sys` (dependensi build ONNX Runtime
dari source membutuhkan pustaka native `re2` yang tidak tersedia di
lingkungan build). Job `laras-build` + pod `debug-find` sudah dihapus.
PVC `laras-build` (30 Gi, berisi hasil kompilasi ORT) dan `laras-data`
(model + ref) **dibiarkan** — bisa dipakai lagi bila rebuild dilanjutkan
suatu saat. **Vonis: jalur rebuild Laras di node VPS ini tertutup**
(bukan lagi "belum diuji" seperti di §3.6 sebelumnya).

#### Pivot — install Laras di sandbox Muse

Biner **prebuilt 0.2.1 (27,6 MB)**, diekstrak dari image
`ghcr.io/urdemonlord/laras-tts:0.2.1` lewat registry API tanpa docker,
**JALAN di sandbox** — CPU sandbox AMD EPYC punya AVX2, jadi masalah ISA
yang membunuh biner yang sama di VPS (§1.11) tidak ada di sini. Model
`anak10thn/pocket-tts-indonesian` yang sama (9 berkas, ±160 MB, sha256
lolos) dipakai ulang.

| Item | Nilai |
| --- | --- |
| Lokasi install (persisten, tahan ganti VM) | `~/workspace/laras-tts/` (193 MB) |
| Biner | `bin/laras` — prebuilt 0.2.1, 27,6 MB |
| Model | `data/models/` — 9 berkas `anak10thn/pocket-tts-indonesian` |
| Referensi voice | `data/ref/` — 4 ref MP3 asli + versi trim 15 detik (`*-ref-15s.mp3`) |
| Server | `http://127.0.0.1:8080`, starter `start-server.sh`, detail `README.md` |
| Verifikasi server | `GET /health` → 200 `{"ok":true,"engine":"laras","version":"0.2.1","lang":"id",...}` |

#### Gate test + uji kualitas — LULUS

Gate test (Zenith, 26 Nov 19:53): gufron, `takes=1`, kalimat yang sama
dengan §1.11 → **WAV 10,40 dtk, 499.244 B — LULUS**. Uji kualitas
(takes=3, teks klip 0 T-013) → **4/4 berhasil**:

| Voice | Durasi WAV | Ukuran |
| --- | --- | --- |
| gufron | 28,00 dtk | 1.344.044 B |
| anak-anak | 29,68 dtk | 1.424.684 B |
| resah | 15,68 dtk | 752.684 B |
| my voice | 6,40 dtk | 307.244 B |

WAV di workspace (`vps-render/voice-tests/test-laras-*-klip0-takes3.wav`
+ `test-laras-gufron-takes1.wav`) — durasi/ukuran terverifikasi via
ffprobe. Catatan: kemiripan suara + kelengkapan kata masih perlu cek
dengar manusia (seperti direncanakan di §1.11 langkah 2).

#### Pelajaran operasional (sandbox)

- **Ref panjang bikin server hang:** ref anak-anak 48,6 dtk → sintesis
  `takes=3` menggantung sampai timeout 300 dtk tanpa output. Trim ke
  15 detik (sesuai syarat cloning 10–15 dtk) → selesai ±55 dtk. **Selalu
  pakai `*-ref-15s.mp3`** untuk anak-anak/resah/my-voice; gufron
  (21,7 dtk) aman.
- **Kecepatan bicara beda jauh antar voice untuk teks yang sama:**
  my-voice 6,4 dtk vs gufron 28,0 dtk — durasi narasi per voice harus
  diukur per voice, jangan diasumsikan sama.
- **Server sempat mati 2× saat host sandbox kehabisan RAM** (available
  sempat 571 MB) — proses hilang tanpa pesan error. Restart pakai
  `start-server.sh`; tidak ada data hilang (model + ref di disk).
- Kecepatan di sandbox (±1,5 shared CPU): RTF ±2–3× — 1 klip ±10 dtk
  ≈ 30 dtk wall `takes=1`, ±45–55 dtk wall `takes=3`.

**Keadaan akhir: narasi kini lewat Laras di sandbox Muse; render tetap
di VPS, tidak berubah.** VoiceStudio laptop tetap ada sebagai cadangan.
Sisa yang menunggu di §3.6 untuk mesin TTS VPS (upgrade RAM / node baru)
tidak lagi mendesak — Laras sandbox adalah jalur produksi narasi saat ini.

### 1.13 Deployment draf office-3d (`muse-hq-office-3d-draft`) — 26 Nov 21:52

Perintah pemilik 26 Nov 21:51: draf kantor 3D (`muse-hq-3d-draft`, Space
Library-only) dideploy juga ke Kubernetes sebagai deployment TERPISAH —
pola sama seperti clone live (§1.4), host baru, hostPath baru. Live tidak
disentuh.

Sumber draf: `~/workspace/ts-spaces/muse-hq-3d-draft/index.html` (2.742
baris, 166.906 byte, md5 `c35e6f2d3efa8b32b9c2164d8bb7bb8d`, judul
"MUSE HQ 3D Draft") + `assets/three.min.js` (603.445 byte, md5
`eb8549863a97355411c3259a3f93b8e1` — sama seperti live).

Yang dibuat (ns `apps`, manifest
`~/workspace/muse-hq/hidden_files/muse-hq-office-3d-draft.yaml`, salinan di
VPS `/home/rlordz12/muse-hq-office-3d-draft.yaml`):

- **hostPath baru** `/home/rlordz12/hq-office-3d-draft` di vps-29 —
  `index.html` + `assets/` disalin via scp lewat tunnel proxy, `chmod 644`
  kedua file. Direktori live `/home/rlordz12/hq-office-3d` tidak disentuh.
- **Deployment `muse-hq-office-3d-draft`** (nginx:alpine, replicas 1,
  `nodeSelector: { kubernetes.io/hostname: vps-29 }` langsung di manifest —
  pelajaran §1.4, jangan ulangi tanpa pin), resources sama seperti live
  (request 50m/64Mi, limit 200m/128Mi), hostPath `readOnly`.
- **Service** `muse-hq-office-3d-draft-service` (ClusterIP :80) +
  **Ingress** `muse-hq-office-3d-draft-ingress` (Traefik websecure,
  cluster-issuer letsencrypt-prod), host
  `office3d-draft.103.77.78.114.sslip.io`, TLS secret
  `muse-hq-office-3d-draft-tls` (Certificate otomatis dari Ingress,
  READY True).
- Pod `muse-hq-office-3d-draft-7b5677d458-lg79d` 1/1 Running, 0 restart,
  di vps-29; rollout sukses.

Verifikasi (26 Nov ~21:55–22:05):

- **Lewat Traefik NodePort** (`curl --resolve …:32557:127.0.0.1` dari
  vps-29): index **HTTP 200, 166.906 B, md5 sama** dengan lokal + VPS,
  judul "MUSE HQ 3D Draft"; `assets/three.min.js` **HTTP 200, 603.445 B,
  md5 sama**. Rantai file → pod → Traefik → klien terverifikasi penuh.
- **Catatan uji:** curl dari VPS ke IP publiknya sendiri
  (`https://office3d…/`) gagal untuk live MAUPUN draf (HTTP 000) — hairpin
  NAT VPS ke IP publiknya rusak secara umum, jadi itu BUKAN alat uji
  yang valid. Uji yang benar: NodePort (di atas) atau dari luar klaster.
- **Live tidak tersentuh:** `https://office3d.103.77.78.114.sslip.io/`
  tetap HTTP 200, md5 `d391e4a3…` (v25); Deployment/Service/Ingress/
  sertifikat live tidak diubah.

**Pelajaran — mode direktori ikut tersalin `scp -r`:** `assets/` di sumber
bermode 770, dan `scp -r` mempertahankannya — nginx (user `nginx` di
container, bukan pemilik) tidak bisa masuk direktori → `three.min.js`
HTTP 403 padahal file-nya sendiri 644. Perbaikan: `chmod 755` pada
direktori `assets/` di VPS, lalu 200. Ini sekelas dengan BUG 5 draf
(aset tidak tersalin → 404): **selalu verifikasi URL aset, bukan hanya
index** — dan samakan mode direktori dengan live (`assets/` live: 775).

**Wiring WS draf (dilaporkan apa adanya, tidak diubah):** JS draf
menunjuk ke hq-ws LIVE (`wss://office3d.103.77.78.114.sslip.io/ws`,
`connectLive`/`applyLiveStatus` verbatim dari live) — draf hanya
MENAMPILKAN status live yang sama, read-only, tidak pernah POST. Jadi
draf dan live berbagi sumber status; tidak ada server WS kedua. Status
"Live" di draf hanya bisa dibuktikan dari HP/browser pemilik (sandbox
tidak bisa verifikasi WS).

**Batasan verifikasi dari sandbox:** egress proxy sandbox tidak bisa
mengambil host baru ini (curl: empty reply; fetch browser: 403) sementara
host live lolos — sisi server terbukti jalan lewat NodePort (di atas).
Cek akhir tampilan + WS "Live" menunggu pemilik membuka
`https://office3d-draft.103.77.78.114.sslip.io/` di HP-nya.

### 1.14 Deployment render standby (`remotion-render`) — 26 Nov 21:49–22:12

Perintah pemilik 26 Nov 21:49: hasil render dinilai kacau — jalur render
diubah dari Job ephemeral per klip menjadi **Deployment khusus render,
standby 1 pod**. Prinsip pemilik 26 Nov 21:57: **"render harus aman di
VPS, bagaimanapun caranya"** — keandalan + isolasi resource didahulukan
(tanpa OOM / mengganggu layanan lain, progres terpantau, timeout bersih);
rute default muted + mix stems, baked video-audio tetap tutup untuk
T-013 (§1.12, §1.14 di bawah).

Yang dibuat (ns `render`, manifest
`~/workspace/muse-hq/hidden_files/vps-render/remotion-render-deployment.yaml`):

| Resource | Nilai |
| --- | --- |
| Deployment | `remotion-render`, replicas 1, **1/1 Running** |
| Node | **vps-29 — BUKAN vps-38**, `nodeSelector: { kubernetes.io/hostname: vps-29 }` |
| Alasan node | PVC `render-work` terikat vps-29 oleh local-path nodeAffinity; pod di vps-38 tidak bisa mount PVC itu (§1.11, §1.13 menemukan hal yang sama untuk hostPath office-3d) |
| Image | `mcr.microsoft.com/playwright:v1.55.0-noble` (sudah ada browser + dependensi) |
| Command | `sleep infinity` — pod idle ~0 CPU/RAM sampai ada render |
| Volume | PVC `render-work` ter-mount di `/work` (proyek Remotion + `out/`) |
| Requests | **1000m / 1536Mi** |
| Limits | **7000m / 2048Mi** (peak render muted terukur ~1383 Mi → margin ~665 Mi) |

Pemicu render: `trigger-render-standby.sh` (di host, detached + log) +
`in-pod-render.sh` (dijalankan di dalam pod via `kubectl exec`) —
render di dalam pod **identik** dengan Job (script yang sama,
`render-cf.cjs`); output tetap di PVC `/work/remotion-poc-byhkab/out/`.
Mode yang didukung: `video` (muted) dan `still`. Mode `video-audio`
**DITOLAK mentah-mentah oleh trigger** (rute baked merangkak — §1.12).

**4 lapis pengaman** (jawaban atas prinsip "render harus aman"):

1. **Resource di-kunci di manifest** — requests 1000m/1536Mi, limits
   7000m/2048Mi; pod standby tidak bisa makan RAM node tanpa batas.
2. **Pre-flight RAM** — sebelum start, trigger cek RAM available node:
   video butuh ≥ **1500 MB**, still ≥ **800 MB**; di bawah itu render
   **ditolak** (exit 2), bukan dibiarkan OOM di tengah jalan.
3. **Serialisasi** — satu render dalam satu waktu: trigger menolak bila
   ada Job render aktif di klaster **dan** bila lock file
   `/tmp/render-standby.lock` masih ada (lock dihapus otomatis saat
   render selesai/gagal; hapus manual hanya bila yakin tidak ada render
   jalan).
4. **Recycle wajib + timeout** — pod **di-delete tiap selesai render**
   (Deployment membuat pengganti yang segar): tiap render membocorkan
   ~2 browser Chrome yatim (~150 MB) yang tidak bisa di-kill dari dalam
   pod, jadi tanpa recycle memori pod merangkak naik tiap render.
   Timeout GNU di dalam pod: **1500 dtk video / 300 dtk still** —
   render yang menggantung dimatikan bersih, bukan dibiarkan jalan
   sampai `activeDeadlineSeconds` Job (pola lama).
5. **Progres terpantau** — log render dibaca dengan `tr '\r' '\n'`
   (pelajaran §1.12: progres Remotion pakai `\r` tanpa `\n`, jadi
   `kubectl logs | tail` biasa menyesatkan — progres tidak kelihatan
   sampai render selesai).

Konteks — kenapa Job lama ditinggalkan (26 Nov ~21:40–22:10):

- Job muted2 klip 0 (+28 px) **merangkak 0 frame ~12 menit** lalu
  **dimatikan 22:10** (terkonfirmasi hilang): node vps-29 swap-thrash —
  RAM 80%, swap 2,8 GB, bebas ~1290 MB. Setelah dimatikan RAM membaik
  80% → 71%, bebas ~1697 MB. Sync PVC +28 px tetap berlaku, tidak perlu
  sync ulang; re-render klip 0 pindah ke pod standby setelah smoke test
  + verdict mask Mira.
- Job audio baked (`remotion-t013-klip0-audio`) **Failed** — rute baked
  resmi ditutup untuk T-013; rakitan final tetap muted + mix stems
  (`mix-cf-final.py`, desain §1.12).
- Node vps-29 saat itu: RAM available **1758 MB**, swap 2602/4095 MB —
  video lolos tipis dari syarat pre-flight 1500 MB. Artinya margin
  render video di vps-29 **sangat tipis**; satu layanan lain yang makan
  RAM bisa membuat render ditolak pre-flight (itu disengaja — lebih baik
  ditolak di awal daripada OOM di tengah).

**Smoke test — BERHASIL 26 Nov 22:11–22:12** (skrip
`run-smoke-watcher.sh`, menunggu sampai tidak ada Job aktif, lalu
memicu still frame 168 lewat pod standby):

- Still frame 168 **< 1 menit** (`npm install` ~11 dtk + render),
  `SMOKE-EXIT=0`.
- PNG **1080×1920, 684.309 byte, md5
  `5a60b58a620b5730c1e6a5671be359ae`** — **identik** di pod, di /tmp VPS,
  dan di sandbox (tiga salinan, md5 sama).
- Peak memori still **~1000 Mi** dari limit 2048 Mi.
- Pod lama didaur ulang setelah smoke, pengganti **Ready** (recycle
  wajib terbukti jalan).
- Still disalin ke folder klip 0 untuk verifikasi mask Mira; video
  re-render klip 0 menunggu verdict mask.

**Perbaikan bug laten — template `remotion-clip-job.yaml`:**
`nodeSelector: { workload: render }` **dihapus** → diganti
`kubernetes.io/hostname: vps-29`. Template Job lama **Pending
selamanya**: label `workload=render` hanya ada di vps-38, sedangkan PVC
`render-work` hanya bisa di-mount di vps-29 — tidak ada node yang
memenuhi kedua syarat sekaligus. **Catatan: template
`render-full-job.yaml` dan `render-smoke-job.yaml` MASIH punya
nodeSelector yang sama (belum diperbaiki)** — jangan dipakai sebelum
diperbaiki dengan cara yang sama.

**Opsi lanjutan (belum dikerjakan, belum diputuskan):** migrasi render
ke **vps-38 (95% kosong)** — butuh PVC baru di vps-38 + salin proyek
406 MB + DNS bypass Opsi B (§1.6) + image pull. Ini menghilangkan
masalah RAM tipis di vps-29; dicatat di §3.6 butir 4.

## 2. Keputusan desain: pisah VoiceStudio dan render Job

Diputuskan pemilik server 26 November 2026 ~01:34 ("Pisah seperti saranmu"):

1. **VoiceStudio = Deployment persistent, replicas 1, di vps-38.** Harus tetap
   hangat: setiap pod baru berarti pull image ~5,6 GiB + load model 3,2 GB.
2. **Render = satu Job per video, ephemeral, TTL cleanup 86400.** Objek yang
   tepat untuk run-once-then-die adalah **Job, bukan Deployment** — Deployment
   me-restart pod yang selesai, jadi bukan objek yang tepat untuk pekerjaan
   sekali jalan.
3. **Satu pod gabungan (voice + render) DITOLAK.** Hitungannya: 3,4 + 1,6 =
   5,0 GiB dari ~7,75 GiB allocatable vps-38 → maksimal 1 render bersamaan,
   kopling crash (satu mati, keduanya mati), skala terkunci.
4. **KEDA tidak dipakai dulu** — render hanya beberapa kali sehari; autoscaling
   event-driven baru dipertimbangkan bila frekuensi render naik.

Perubahan 26 Nov 21:57 (§1.14): pemilik membalik butir 2 — render kini
Deployment standby `remotion-render` (replicas 1, `sleep infinity`),
bukan Job per video. Alasannya: hasil render dinilai kacau + Job muted
merangkak 0 frame di node vps-29 yang swap-thrash; prinsip baru "render
harus aman di VPS, bagaimanapun caranya". Job tetap ada sebagai
fallback/pembanding, dan template `remotion-clip-job.yaml` diperbaiki
pin-nya ke vps-29 (§1.14); dua template lain masih ber-nodeSelector lama.

## 3. Yang MENUNGGU tangan / keputusan pemilik server

> Bagian ini **bukan** yang sudah selesai. Jangan dianggap beres sebelum ada
> verifikasi baru.

### 3.1 Terima 2 lisensi model — SELESAI 26 Nov 09:36:30

Diterima **pemilik server sendiri** (bukan agen — menerima lisensi adalah
**persetujuan legal**, agen tidak boleh menerimanya atas nama pemilik):

- `k2-fsa/OmniVoice` — lisensi **CC-BY-NC**,
  fingerprint `v2:d938ea928581624d5732507057d8530320f638ac61426165cbd5abe67be5c9bc`
- `eustlb/higgs-audio-v2-tokenizer`,
  fingerprint `v2:6b442e435fd392f7726e7c44a30da71bba23adecaabe4c8da725cb30579ecff1`

Verifikasi: keduanya `accepted: true`,
`accepted_at: 2026-11-26T05:36:30Z`, fingerprint cocok. Efeknya:
`POST /v1/audio/speech` tidak lagi 403 `model_licence_required` — bloker
berikutnya adalah unduhan weights (§1.6) dan batas eksekusi CPU (§1.9).
Lisensi bersifat **per-instance** (lisensi laptop tidak berlaku untuk VPS).

### 3.2 Impor voice dari laptop — SELESAI 26 Nov ~02:28–02:32

4 voice laptop terimpor ke VPS. Rutenya (terverifikasi saat aplikasi desktop
laptop jalan): ekspor via `GET /profiles` + `GET /profiles/{id}/audio`
(satu file referensi MP3 + `ref_text` per voice), impor via `POST /profiles`
multipart (`name`, `ref_audio=@file.mp3`, `ref_text`,
`language=Indonesian`).

| Voice | ID di VPS |
| --- | --- |
| gufron | `4b11bf25` |
| anak-anak | `50290a19` |
| resah | `2d15d02a` |
| my voice | `8cfb0d78` |

Verifikasi: `GET /v1/audio/voices` → **18 voice total** (4 impor + demo0001
+ 13 alias OpenAI). File referensi MP3 tersimpan di workspace
(`voice-export/`). Hasil test generate per voice ada di §1.9.

### 3.3 Secret `voicestudio-api-key` — DIPUTUSKAN pemilik 26 Nov 02:24

Secret di klaster berisi teks placeholder contoh perintah
(`<kunci-pilihan-user>`), dan **pemilik menerima placeholder itu sebagai
kunci** — jangan dibuat ulang tanpa perintahnya. GitHub secret
`VOICESTUDIO_API_KEY` (dibuat 26 Nov 01:46, terverifikasi ada via API)
harus berisi string yang **sama persis**; kalau beda, server menolak
dengan 401.

Bila suatu saat Secret diganti, pod harus di-restart supaya membaca env baru
(env dari `secretKeyRef` hanya dibaca saat pod start — Secret yang diubah
tidak mengubah pod yang sudah jalan):

```bash
sudo /usr/local/bin/kubectl rollout restart deploy/voicestudio -n render
sudo /usr/local/bin/kubectl rollout status deploy/voicestudio -n render
```

> ⚠️ **Jangan pernah tempel nilai kunci di chat, dan jangan pernah commit ke
> repo.** Kunci diketik langsung di server; GitHub secret yang sama isinya harus
> sama persis (tanpa spasi di depan/belakang) — kalau beda, server menolak
> dengan 401.

### 3.4 GitHub secret `K8S_RENDER_TOKEN` + persetujuan diff workflow

- Pemilik membaca token SA sendiri dengan perintah di §1.2, lalu menambahkannya
  sebagai GitHub secret `K8S_RENDER_TOKEN` di repo `birunix-render`
  (Settings → Secrets and variables → Actions → New repository secret).
- Diff workflow (langkah TTS ke VoiceStudio + whisper onset + rebuild timing)
  sudah disiapkan sebagai file diff di workspace dan lolos `git apply --check`,
  tapi **belum di-push — menunggu persetujuan pemilik** sebelum workflow
  diubah. Guard yang disiapkan di diff: step TTS tanpa hook whisper→Remotion
  harus FAIL (hook itu milik produksi), dan whisper yang gagal harus FAIL
  (jangan lanjut dengan timing lama).

### 3.5 Keputusan rute runner → API server + eksposur VoiceStudio

Dua keputusan yang saling terkait, keduanya belum diputuskan:

1. **Rute runner → API server** (§1.5: IP publik belum terbukti). Opsi:
   (a) diagnosa kenapa koneksi 6443 dari internet ditutup tanpa respons HTTP,
   lalu perbaiki; (b) kubeconfig runner memakai alamat Tailscale
   `100.123.155.8` — perlu cek SAN dulu (sertifikat tidak memuat alamat ini);
   (c) SSH tunnel / port-forward dari runner ke API server. Mana pun yang
   dipilih, `--insecure-skip-tls-verify` tetap dilarang.
2. **Eksposur VoiceStudio untuk runner.** Service sekarang ClusterIP (hanya
   dari dalam klaster). Opsi: Ingress publik + TLS + Bearer, atau rute
   SSH/port-forward. Keputusan ini menentukan endpoint final langkah TTS di
   workflow (§3.4) — jangan kunci endpoint di workflow sebelum diputuskan.

### 3.6 Keputusan lanjut: mesin TTS VPS + perbaikan akar jaringan vps-38

Menunggu keputusan pemilik (setelah §1.9 dan §1.10):

1. **Mesin TTS VPS — jalur GGUF SUDAH TERJAWAB (§1.10), jalur Laras
   SUDAH TERJAWAB (§1.11), rebuild Laras SUDAH TERJAWAB (§1.12): tidak
   bisa di node ini. Narasi kini lewat Laras di sandbox Muse (§1.12) —
   pivot pemilik 26 Nov 18:49 — jadi pertanyaan mesin TTS VPS tidak lagi
   mendesak.** Opsi (a) engine ringan GGUF / Supertonic-3 gugur — bukan
   soal engine, tapi CPU Xeon E5620 tanpa AVX dan RAM 7,75 GB. Laras
   (§1.11) gugur di ISA yang sama: biner ONNX Runtime prebuilt butuh AVX2
   (SIGILL, exit 132) — upgrade RAM tidak menolong Laras. Opsi yang
   tersisa:
   (1) **Laras di sandbox Muse (§1.12)** — jalur narasi yang dipakai
   sekarang; (2) **narasi via VoiceStudio laptop (RTX 3050, terbukti
   jalan)** — cadangan bila sandbox bermasalah; (3) **upgrade RAM VPS** —
   dots / VoxCPM2 baru bisa LOAD, tapi kecepatan di E5620 kemungkinan
   tetap terlalu lambat untuk narasi 60 dtk; (4) **node baru dengan AVX2
   + RAM lebih besar** — membuka jalur GGUF cepat dan jalur Laras di VPS;
   (5) **rebuild Laras dari source dengan `target-cpu=x86-64` — SUDAH
   DIUJI, GAGAL (§1.12)**: Job `laras-build` Failed 149 menit, error
   `could not find native static library 're2'` saat `cargo build`
   `ort-sys`; (6) **cloud voice cloning** — riset 26 Nov 13:18:
   ElevenLabs Starter $6/bln (IVC, 30 rb kredit) atau Fish Audio
   (cloning gratis, Plus $15/bln), ref MP3 yang ada langsung bisa dipakai
   di Fish; keputusan dari uji dengar. Opsi (c) buru budget override
   600 dtk dan (d) torch-compile tetap tidak layak / payoff tak pasti
   seperti sebelumnya.
2. **Perbaikan akar jaringan pod vps-38** (§1.6): pod-to-pod tetap rusak;
   perbaikan butuh akses pemilik ke vps-38 (cek `iptables FORWARD`,
   `bridge fdb`, `ip neigh`) atau persetujuan eksplisit untuk rute
   privileged. Sampai diperbaiki, Job render di vps-38 yang butuh DNS /
   pod-to-pod kena masalah yang sama — DNS bypass per-pod (§1.6) adalah pola
   yang dipakai bila ada workload lain yang butuh nama eksternal.
3. **Lisensi Serveurperso/OmniVoice-GGUF — MOOT, JANGAN diterima dulu.**
   Model `Serveurperso/OmniVoice-GGUF`, lisensi **CC-BY-NC-4.0**,
   fingerprint `v2:c61e2d14574712db2441bc5e7e01eb5cec871473fcf95a8e8ca07799fa0d8f42`
   — statusnya MOOT sampai ada keputusan soal jalur GGUF (item 1 di atas).
   Menerima lisensi adalah persetujuan legal — agen tidak boleh menerimanya
   atas nama pemilik.
4. **Migrasi render ke vps-38 (opsi, belum diputuskan)** (§1.14):
   vps-38 95% kosong, sedangkan render video di vps-29 lolos tipis dari
   syarat pre-flight RAM 1500 MB (available 1758 MB). Migrasi butuh PVC
   baru di vps-38 + salin proyek 406 MB + DNS bypass Opsi B (§1.6) +
   image pull. Keputusan menunggu pemilik; sampai diputuskan, render
   tetap di pod standby vps-29.

## 4. Ringkasan

- **VoiceStudio jalan di vps-38** (Deployment + PVC 10Gi + Service :3900,
  pod 1/1 Running, `/health` 200, DNS bypass §1.6, STT faster-whisper-base
  §1.7, profil TTS `tier=fast` §1.8) — lisensi diterima (§3.1), 4 voice
  diimpor (§3.2), weights terunduh. **Tapi TTS belum bisa dipakai produksi:**
  test generate gufron → HTTP 503 di 600,28 dtk, CPU VPS ~165x real-time
  (§1.9). Percobaan engine ringan GGUF (§1.10) juga gagal — node Xeon E5620
  tanpa AVX, RAM 7,75 GB: dots-tts dan VoxCPM2 OOM, biner GGUF menolak jalan;
  vonis: tidak ada engine TTS Indonesia yang bisa jalan di node ini. Uji Laras
  TTS (§1.11) juga gugur di hardware yang sama — biner ONNX Runtime prebuilt
  butuh AVX2 (SIGILL, exit 132); model ±160 MB + 4 ref MP3 tersimpan di PVC
  `laras-data` 3Gi (masih ada, PV terikat vps-38). Rebuild Laras dari source
  di VPS juga GAGAL (§1.12: Job `laras-build` Failed 149 menit, error
  `could not find native static library 're2'`). **Pivot pemilik 26 Nov
  18:49: Laras JALAN di sandbox Muse** — biner prebuilt 0.2.1 jalan (CPU
  AMD EPYC punya AVX2), gate test + uji kualitas 4/4 voice lulus (§1.12);
  render tetap di VPS. Narasi kini lewat Laras sandbox; VoiceStudio laptop
  tetap cadangan; alternatif cloud (ElevenLabs/Fish) ada di §3.6.
- **Jaringan pod vps-38 rusak di data plane** (§1.6): pod-to-pod mati, DNS
  via kube-dns mati; bypass DNS per-pod adalah workaround, bukan perbaikan
  akar.
- **Runner GitHub punya akses minimal** (SA + Role hanya Job/Pod di ns `render`)
  — token dibaca pemilik sendiri untuk `K8S_RENDER_TOKEN` (§3.4).
- **Penjadwalan diperbaiki di dua tempat:** 3 template Job render di-pin ke
  vps-38 via `workload: render`; office-3d di-pin ke vps-29 karena hostPath
  (§1.3, §1.4).
- **Draf office-3d punya Deployment sendiri** (§1.13):
  `muse-hq-office-3d-draft` di `office3d-draft.103.77.78.114.sslip.io`,
  hostPath baru di vps-29, juga di-pin ke vps-29; klien WS-nya menunjuk ke
  hq-ws live (read-only, status yang sama dengan live).
- **Render kini Deployment standby** (§1.14): `remotion-render`
  (replicas 1, `sleep infinity`, di vps-29 karena PVC `render-work`
  terikat vps-29), pemicu `trigger-render-standby.sh` + `in-pod-render.sh`;
  4 lapis pengaman (requests/limits, pre-flight RAM, serialisasi + lock,
  recycle pod + timeout, progres `tr '\r' '\n'`); rute baked video-audio
  ditutup untuk T-013; smoke test still frame 168 lulus 26 Nov 22:11–22:12
  (< 1 menit, md5 identik di pod/VPS/sandbox). Template
  `remotion-clip-job.yaml` diperbaiki pin-nya ke vps-29; dua template lain
  masih ber-nodeSelector lama.
- **Rute API publik belum terbukti** (§1.5) — keputusan rute + eksposur
  VoiceStudio menunggu pemilik (§3.5).
- Image VoiceStudio terukur **~5,6 GiB** — angka 17 GB di docs/02 sudah tidak
  dipakai.
