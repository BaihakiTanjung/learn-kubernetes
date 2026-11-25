# VoiceStudio, RBAC Runner & Perbaikan Penjadwalan

Status: **Deployment VoiceStudio + SA/RBAC runner + nodeSelector template + pin
office-3d SELESAI 26 November 2026 ~01:43–02:09**; tiga hal yang MENUNGGU tangan
pemilik server dipisah tegas di §3 — jangan dicampur dengan yang sudah selesai.
Tidak ada perubahan di klaster pada penulisan dokumen ini; semua yang di bawah
adalah yang benar-benar terjadi dan terverifikasi pada jam tersebut.

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
| Env | `OMNIVOICE_API_KEY` dari Secret `voicestudio-api-key` (`secretKeyRef`) |
| Pod | 1/1 Running di **vps-38** |
| Limit memori | 4 GiB (peak generate terukur 3,4 GiB) |

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
        resources:
          limits:
            memory: 4Gi
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

**Koreksi angka:** image VoiceStudio terukur **~5,6 GiB**, bukan 17 GB seperti
perkiraan di [docs/02-garbage-collection-optimasi.md](02-garbage-collection-optimasi.md).
Angka 17 GB itu perkiraan sebelum image ter-pull dan **tidak lagi dipakai** —
termasuk untuk hitungan prune di docs/02 §3A (image VoiceStudio tetap ikut
terhapus bila di-prune, karena Deployment-nya tidak selalu punya pod yang jalan,
tapi pull ulangnya ~18 menit, bukan "10+ menit untuk 17 GB").

**Yang BELUM bisa dilakukan** meski servernya sudah jalan: generate suara
(`POST /v1/audio/speech` → 403, §3.1) dan impor 3 voice dari laptop (§3.2).

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

### 3.1 Terima 2 lisensi model (keputusan pemilik server, bukan agen)

`POST /v1/audio/speech` masih gagal:

```text
HTTP 403 model_licence_required
```

Dua model belum diterima lisensinya:

- `k2-fsa/OmniVoice` — lisensi **CC-BY-NC**
- `eustlb/higgs-audio-v2-tokenizer`

Menerima lisensi adalah **persetujuan legal** — agen tidak boleh menerimanya
atas nama pemilik. Pemilik menerimanya sendiri di Model Catalogue (Hugging
Face), lalu test generate diulang. Sampai lisensi diterima, `/health` dan
`/v1/audio/voices` jalan, tapi **tidak ada suara yang bisa di-generate**.

### 3.2 Buka aplikasi desktop VoiceStudio di laptop

Impor 3 voice (`my voice`, `resah`, `anak-anak`) terblokir karena aplikasi
desktop VoiceStudio di laptop **tidak berjalan**: `GET /profiles` via tunnel
proxy 3130 gagal 5x.

Langkah pemilik: buka aplikasinya di laptop, pastikan laptop menyala, lalu
retry impor. Rutenya (sudah terverifikasi saat aplikasi jalan):

- Ekspor dari laptop: `GET /profiles` + `GET /profiles/{id}/audio`
  (satu file referensi MP3 ≤20 detik + `ref_text` per voice).
- Impor ke VPS: `POST /profiles` multipart (`name`, `ref_audio=@file.mp3`,
  `ref_text`, `language=Indonesian`).
- Verifikasi: `GET /v1/audio/voices` + 1 test generate pendek per voice.

### 3.3 Buat ulang Secret `voicestudio-api-key` dengan kunci asli

Secret di klaster saat ini **berisi teks placeholder contoh perintah**
(`<kunci-pilihan-user>`), bukan kunci asli — pod yang jalan sekarang memakai
kunci placeholder itu. Pemilik membuat ulang Secret dengan kunci pilihannya,
lalu menyamakannya dengan GitHub secret `VOICESTUDIO_API_KEY` (sudah dibuat
26 Nov 01:46, terverifikasi ada via API — nilainya tidak bisa diverifikasi,
kecocokannya terbukti lewat test generate nanti).

```bash
# di vps-29 — ganti <kunci-pilihan-user> dengan kunci yang dipilih
sudo /usr/local/bin/kubectl create secret generic voicestudio-api-key -n render \
  --from-literal=api-key='<kunci-pilihan-user>' \
  --dry-run=client -o yaml | sudo /usr/local/bin/kubectl apply -f -
```

Setelah Secret diganti, pod harus di-restart supaya membaca env baru
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

## 4. Ringkasan

- **VoiceStudio jalan di vps-38** (Deployment + PVC 10Gi + Service :3900,
  pod 1/1 Running, `/health` 200) — tapi **belum bisa generate suara**:
  lisensi 2 model belum diterima (§3.1), 3 voice belum diimpor karena aplikasi
  laptop tidak jalan (§3.2), dan Secret klaster masih berisi placeholder (§3.3).
- **Runner GitHub punya akses minimal** (SA + Role hanya Job/Pod di ns `render`)
  — token dibaca pemilik sendiri untuk `K8S_RENDER_TOKEN` (§3.4).
- **Penjadwalan diperbaiki di dua tempat:** 3 template Job render di-pin ke
  vps-38 via `workload: render`; office-3d di-pin ke vps-29 karena hostPath
  (§1.3, §1.4).
- **Rute API publik belum terbukti** (§1.5) — keputusan rute + eksposur
  VoiceStudio menunggu pemilik (§3.5).
- Image VoiceStudio terukur **~5,6 GiB** — angka 17 GB di docs/02 sudah tidak
  dipakai.
