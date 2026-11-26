# VoiceStudio, RBAC Runner & Perbaikan Penjadwalan

Status: **Deployment VoiceStudio + SA/RBAC runner + nodeSelector template + pin
office-3d SELESAI 26 November 2026 ~01:43–02:09; lisensi diterima + voice
diimpor + DNS bypass + STT + profil performa SELESAI 26 Nov ~09:36–11:46;
hasil GGUF SELESAI 26 Nov (§1.10) — node VPS tidak bisa menjalankan engine
TTS Indonesia apa pun (Xeon E5620 tanpa AVX, RAM 7,75 GB).**
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
di §1.10, opsi lanjut di §3.6.

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

1. **Mesin TTS VPS — jalur GGUF SUDAH TERJAWAB (§1.10): tidak bisa di node
   ini.** Opsi (a) engine ringan GGUF / Supertonic-3 gugur — bukan soal
   engine, tapi CPU Xeon E5620 tanpa AVX dan RAM 7,75 GB. Opsi yang tersisa:
   (1) **narasi via VoiceStudio laptop (RTX 3050, terbukti jalan)** — jalur
   yang dipakai sampai ada keputusan lain; (2) **upgrade RAM VPS** —
   dots / VoxCPM2 baru bisa LOAD, tapi kecepatan di E5620 kemungkinan tetap
   terlalu lambat untuk narasi 60 dtk; (3) **node baru dengan AVX2 + RAM
   lebih besar** — membuka jalur GGUF cepat. Opsi (c) buru budget override
   600 dtk dan (d) torch-compile tetap tidak layak / payoff tak pasti seperti
   sebelumnya.
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

## 4. Ringkasan

- **VoiceStudio jalan di vps-38** (Deployment + PVC 10Gi + Service :3900,
  pod 1/1 Running, `/health` 200, DNS bypass §1.6, STT faster-whisper-base
  §1.7, profil TTS `tier=fast` §1.8) — lisensi diterima (§3.1), 4 voice
  diimpor (§3.2), weights terunduh. **Tapi TTS belum bisa dipakai produksi:**
  test generate gufron → HTTP 503 di 600,28 dtk, CPU VPS ~165x real-time
  (§1.9). Percobaan engine ringan GGUF (§1.10) juga gagal — node Xeon E5620
  tanpa AVX, RAM 7,75 GB: dots-tts dan VoxCPM2 OOM, biner GGUF menolak jalan;
  vonis: tidak ada engine TTS Indonesia yang bisa jalan di node ini. Narasi
  tetap lewat VoiceStudio laptop sampai ada keputusan mesin baru (§3.6).
- **Jaringan pod vps-38 rusak di data plane** (§1.6): pod-to-pod mati, DNS
  via kube-dns mati; bypass DNS per-pod adalah workaround, bukan perbaikan
  akar.
- **Runner GitHub punya akses minimal** (SA + Role hanya Job/Pod di ns `render`)
  — token dibaca pemilik sendiri untuk `K8S_RENDER_TOKEN` (§3.4).
- **Penjadwalan diperbaiki di dua tempat:** 3 template Job render di-pin ke
  vps-38 via `workload: render`; office-3d di-pin ke vps-29 karena hostPath
  (§1.3, §1.4).
- **Rute API publik belum terbukti** (§1.5) — keputusan rute + eksposur
  VoiceStudio menunggu pemilik (§3.5).
- Image VoiceStudio terukur **~5,6 GiB** — angka 17 GB di docs/02 sudah tidak
  dipakai.
