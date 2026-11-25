# Garbage Collection + Optimasi Klaster k3s

Status: **assessment + pembersihan level-API SELESAI 26 November 2026 ~01:16–01:20**; perubahan
level-node (prune image, flag kubelet GC, etcd) **MENUNGGU tangan pemilik server**; desain
CronJob GC di §5 adalah **PROPOSAL, menunggu persetujuan**. Tidak ada restart service/workload
yang dilakukan. Tidak ada pod privileged yang dibuat.

## 1. Hasil assessment (26 Nov 2026, ~01:16–01:20, via kubectl read-only)

| Ukuran | vps-29 (control-plane + etcd) | vps-38 (worker) |
| --- | --- | --- |
| Disk total / terpakai / bebas | 123,75 GB / 96,19 GB (78%) / 27,54 GB | 123,75 GB / 3,47 GB (3%) / 120,26 GB |
| Image store (imagefs) | **45,09 GB** (36% dari disk) | 0,09 GB |
| Memori terpakai | 5.986–6.101 Mi (**75–76%**), bebas ~2,06 GB | 303 Mi (3%), bebas ~8,0 GB |
| CPU | 20–39% (naik-turun) | 0% |
| Kondisi node | Ready; MemoryPressure/DiskPressure/PIDPressure semua False | Ready; semua False |
| Inode | 12,4 jt bebas (aman) | 14,8 jt bebas (aman) |

Temuan utama:

1. **Disk vps-29 di 78%, dan 45 GB di antaranya adalah image container.** Ini satu-satunya
   penumpukan besar. Penyebabnya bukan satu image: VoiceStudio 17 GB + Playwright + semua image
   Rancher/fleet/cert-manager/traefik + image lama dari install. kubelet image GC default
   (high 85% / low 80%) **tidak akan pernah jalan** pada kondisi ini, karena ambangnya dihitung
   dari pemakaian imagefs (sekarang 36%) — GC baru aktif bila imagefs melewati 85%.
   Artinya: **threshold saja tidak membersihkan 45 GB yang sudah ada**; perlu prune manual
   satu kali (§3A), lalu threshold baru menjaga supaya tidak menumpuk lagi (§3B).
2. **Memori vps-29 di 75–76%.** Konsumen terbesar: pod `rancher` **2.317 Mi tanpa resource
   limit sama sekali** (spec resources `{}`). Seluruh stack Rancher (rancher + webhook +
   fleet-controller + gitjob + helmops + fleet-agent + capi + turtles + system-upgrade-controller)
   ≈ **2,7 GB**. Sisanya kecil-kecil (hermes3d 150 Mi, 9router 127 Mi, traefik 47 Mi, dst).
   Tidak ada MemoryPressure (ambang eviction kubelet 100 Mi available; tersedia 2,06 GB), tapi
   ruang gerak sempit: satu Job render (peak 1,6 GB) + VoiceStudio (peak 3,4 GB bila nanti
   di-scale) tidak muat bersamaan — sesuai rencana, render pindah ke vps-38 (§4A) dan RAM
   akan ditambah (rencana pemilik server).
3. **vps-38 praktis kosong** (3% disk, 3% memori). Satu-satunya pod di sana: svclb-traefik
   (DaemonSet, otomatis ikut saat join).
4. **Tidak ada pod Failed, tidak ada CrashLoop.** Event join vps-38 yang sempat Warning
   (`CIDRAssignmentFailed`, `FailedCreatePodSandBox` flannel `subnet.env` belum ada,
   `InvalidDiskCapacity` image filesystem) semuanya **transien saat join dan sudah pulih
   sendiri** — node Ready, NoDiskPressure. Tidak perlu tindakan.
5. **Job template sudah bagus:** `render-full-job.yaml`, `remotion-clip-job.yaml`,
   `render-smoke-job.yaml` sudah punya `ttlSecondsAfterFinished: 86400`.
   **Pengecualian: `remotion-smoke-job.yaml` TIDAK punya TTL** — sudah diperbaiki (§2).
6. CronJob `fleet-cleanup-gitrepo-jobs` sudah `successfulJobsHistoryLimit: 0` (minimal).
   `rke2-machineconfig-cleanup-cronjob` tadinya 3 — sudah dipatch ke 1 (§2).

## 2. Yang SUDAH diterapkan (via kubectl/API saja — tanpa restart apa pun)

**Dihapus — Job yang sudah selesai (output sudah terverifikasi & tercatat sebelumnya):**

- `render/remotion-smoke-1` (51 mnt, Complete; MP4 terverifikasi ffprobe, sha256 tercatat)
- `render/render-smoke-1` (77 mnt, Complete; 10 frame terverifikasi)
- `kube-system/helm-install-traefik`, `kube-system/helm-install-traefik-crd` (24 hari)

**Dihapus — pod completed tanpa Job:**

- `cert-manager/cert-check`, `cert-manager/dns-check` (5 hari)

**Dipatch:**

- CronJob `fleet-local/rke2-machineconfig-cleanup-cronjob`: `successfulJobsHistoryLimit` 3 → **1**.
  Controller langsung menegakkannya: 2 Job lama terhapus otomatis, tersisa 1 (yang terbaru, 21 jam).

**Diperbaiki di template (file, bukan klaster):**

- `remotion-smoke-job.yaml`: ditambah `ttlSecondsAfterFinished: 86400` supaya Job smoke
  berikutnya auto-hapus dalam 24 jam (sebelumnya menumpuk selamanya). Ini satu-satunya
  template Job yang belum punya TTL; 3 template lain sudah punya.

**⚠️ Pelajaran — JANGAN hapus Job `helm-install-*`:**

Dua Job helm-install yang dihapus **dibuat ulang sendiri oleh k3s helm-controller dalam
~35 detik** (Job baru `helm-install-traefik` Complete 28 dtk, `helm-install-traefik-crd`
Complete 45 dtk). Helm install-nya idempotent dan traefik tidak terganggu (pod traefik tetap
Running 24 hari, tanpa restart) — tapi menghapusnya sia-sia. **Biarkan Job itu apa adanya.**

**Angka sebelum → sesudah (terukur):**

| | Sebelum | Sesudah |
| --- | --- | --- |
| Job completed | 7 | 3 (1 fleet-local + 2 helm-install yang dibuat ulang controller) |
| Pod Completed | 7 | 3 (1 fleet-local + 2 helm-install baru) |
| Pod Failed | 0 | 0 |
| Memori vps-29 | 75% (5.986 Mi) | 76% (6.101 Mi) — **tidak berubah** |
| Disk vps-29 | 78% (27,54 GB bebas) | 78% (tidak berubah) |

Jujur saja: menghapus pod/Job completed **hanya membersihkan objek API/etcd, bukan RAM atau
disk**. Efeknya kecil tapi nyata untuk kebersihan etcd. Disk 45 GB image store TIDAK BISA
disentuh lewat kubectl — itu butuh §3.

## 3. Yang MENUNGGU tangan pemilik server (belum dijalankan)

> Perintah di bagian ini dijalankan langsung di node (butuh sudo). Akses sudo di vps-29
> terbatas ke kubectl, jadi langkah ini dikerjakan pemilik server sendiri — sama seperti saat
> mengambil node-token di [panduan tambah worker](01-tambah-worker-node.md).

### 3A. Prune image satu kali (vps-29 saja; vps-38 hampir kosong)

```bash
# Lihat dulu isi image store (45 GB itu apa saja):
sudo crictl images
# Bila crictl protes soal endpoint, pakai:
# sudo crictl --runtime-endpoint unix:///run/k3s/containerd/containerd.sock images

# Hapus SEMUA image yang tidak terpakai (tidak direferensikan pod/container apa pun):
sudo crictl rmi --prune
```

- Ekspektasi: reclaim **belum bisa dipastikan sebelum melihat daftar image** — perkiraan
  10–30 GB (image lama Rancher/Playwright/helm-install yang sudah tidak dipakai).
- `rmi --prune` TIDAK menghapus image yang sedang dipakai pod yang jalan — aman untuk
  workload live.
- **Catatan VoiceStudio (17 GB):** karena Deployment-nya `replicas: 0` dan tidak ada pod yang
  memakainya, prune AKAN menghapusnya. Pull ulang 17 GB butuh waktu (~10+ menit) saat nanti
  dibutuhkan setelah upgrade RAM. Bila mau dipertahankan, JANGAN `--prune`; hapus selektif:
  `sudo crictl images` → catat IMAGE ID yang mau dibuang → `sudo crictl rmi <id>` per image.
- Setelah prune, verifikasi: `sudo crictl images | wc -l` dan `df -h /` (bebas harus naik),
  lalu dari mana saja: `kubectl get nodes` tetap Ready.
- **Jangan jalankan prune saat Job render sedang berjalan** — image Playwright yang sedang
  dipakai memang terlindungi, tapi jangan ambil risiko.

### 3B. Flag kubelet GC — config permanen (vps-29 DAN vps-38)

Di masing-masing node, edit `/etc/rancher/k3s/config.yaml`. **Lihat dulu isinya**
(`sudo cat /etc/rancher/k3s/config.yaml`) lalu TAMBAHKAN blok ini — jangan timpa file bila
sudah ada key lain (mis. `--tls-san` nanti):

```yaml
kubelet-arg:
  - "image-gc-high-threshold=70"
  - "image-gc-low-threshold=60"
  - "minimum-image-ttl-duration=2h"
  - "container-log-max-size=10Mi"
  - "container-log-max-files=5"
```

Alasan nilai ini:

- Default kubelet: high 85 / low 80. Dengan imagefs 36% sekarang, GC tidak akan aktif
  sampai disk hampir penuh. **70/60** membuat GC mulai bekerja saat imagefs melewati 70%
  (~86 GB) dan berhenti di 60% — jauh sebelum ambang eviction kubelet (nodefs available
  <10%). Ini "memasang GC"-nya: tanpa ini, tidak ada pembersihan image otomatis yang
  berarti.
- `minimum-image-ttl-duration=2h` = default kubelet, ditulis eksplisit supaya terlihat:
  image yang tidak dipakai >2 jam boleh di-GC.
- `container-log-max-size/files` = default kubelet (10 Mi × 5 file = maks 50 MB log per
  container), ditulis eksplisit supaya terlihat; log container tidak akan membengkak.
- **Tidak diubah (sudah agresif dari default):** container GC (`maximum-dead-containers-per-
  container=1`, `minimum-container-ttl-duration=0s`) — container mati sudah langsung
  dibersihkan kubelet; tidak ada yang perlu diperbaiki.

Terapkan + restart (lakukan saat TIDAK ada Job render aktif):

```bash
# di vps-29 (control-plane + etcd):
sudo systemctl restart k3s
# Dampak: API server down ~30-60 detik, node sempat NotReady sebentar.
# Container yang jalan TIDAK dimatikan (kubelet restart tidak membunuh pod).
# Kanban/hq-ws/office-3d/Postgres tetap jalan; hanya API yang sebentar tidak bisa diakses.

# di vps-38 (worker):
sudo systemctl restart k3s-agent
# Dampak: lebih ringan — tidak ada control-plane di sini; pod di vps-38 tetap jalan,
# node sempat NotReady sebentar.
```

Verifikasi sesudah restart (dari vps-29):

```bash
sudo /usr/local/bin/kubectl get nodes -o wide
# kedua node harus Ready
sudo /usr/local/bin/kubectl top nodes
# angka memori kembali normal
sudo journalctl -u k3s -n 30 --no-pager        # vps-29
sudo journalctl -u k3s-agent -n 30 --no-pager  # vps-38 (dijalankan di vps-38)
# pastikan tidak ada error baru
```

### 3C. Cek ukuran etcd + snapshot (vps-29 saja)

```bash
sudo du -sh /var/lib/rancher/k3s/server/db/etcd
sudo du -sh /var/lib/rancher/k3s/server/db/snapshots
sudo /usr/local/bin/k3s etcd-snapshot list
```

- Bila `etcd` di bawah ~200 MB: tidak ada yang perlu dilakukan (klaster kecil, wajar).
- Bila snapshot menumpuk melebihi retensi (default 5):
  `sudo /usr/local/bin/k3s etcd-snapshot prune`
- Bila DB etcd >500 MB: jangan defrag sendiri — defrag butuh etcdctl + sertifikat etcd,
  siapkan perintahnya setelah melihat ukuran (jangan asal tebak path sertifikat).

### 3D. Bersihkan journal log (kedua node, satu kali, aman)

```bash
sudo journalctl --vacuum-size=500M
sudo journalctl --disk-usage
```

## 4. Keputusan yang MASIH MENUNGGU (bukan yang sudah selesai)

### 4A. Pindahkan Job render ke vps-38 (disarankan — langkah berikutnya yang sudah direncanakan)

Tambahkan ke pod template `remotion-clip-job.yaml` dan `render-full-job.yaml`:

```yaml
      nodeSelector:
        workload: render
```

Efek: Job render (peak 1,6 GB) lari ke vps-38 yang memorinya 97% kosong, vps-29 lega
~1,6 GB selama render. Tidak diterapkan di tugas ini — penjadwalan render adalah langkah
terpisah.

### 4B. Batasi memori Rancher (opsi, butuh persetujuan — restart pod Rancher ~1 menit)

Rancher sekarang tanpa limit dan memakai 2,3 GB. Bila mau dibatasi:

```bash
sudo -n /usr/local/bin/kubectl set resources deploy/rancher -n cattle-system \
  --requests=memory=1Gi,cpu=250m --limits=memory=3Gi,cpu=2000m
```

Risiko: limit 3 Gi memberi headroom ~700 MB dari pemakaian sekarang; bila Rancher melonjak
saat operasi berat (upgrade/provisioning), bisa OOMKill dan restart sendiri. Bila itu
terjadi, naikkan limitnya. **Jangan set limit di bawah 2,5 Gi.**

### 4C. Hapus Rancher seluruhnya (opsi besar — keputusan pemilik server, bukan rekomendasi otomatis)

Membebaskan ~2,7 GB (seluruh stack Rancher + fleet + webhook + capi + turtles). Klaster
tetap jalan penuh via kubectl; yang hilang hanya UI Rancher. Tidak bisa dibatalkan dengan
mudah (install ulang). Pertimbangkan hanya bila UI Rancher memang jarang dipakai.

### 4D. Yang TIDAK perlu diubah

- Container GC kubelet: sudah agresif (default).
- Log container: sudah dibatasi default 10 Mi × 5.
- CronJob `fleet-cleanup-gitrepo-jobs`: sudah history 0.
- Taint/label node: sudah benar (`workload=render` di vps-38, tanpa taint).

## 5. Desain CronJob GC privileged (PROPOSAL — TIDAK diterapkan, menunggu persetujuan)

**Kapan ini dibutuhkan:** hanya bila pemilik server TIDAK mau mengubah config node (§3B)
tapi tetap mau prune image otomatis berkala. **Rekomendasi: pakai §3A + §3B saja** — lebih
sederhana, tanpa workload privileged. CronJob di bawah ini adalah cadangan.

**Cara kerja:** seminggu sekali (Minggu 03:00), sebuah Job menjalankan `crictl rmi --prune`
dengan socket containerd node di-mount. Karena satu CronJob hanya jalan di satu node,
manifest ini harus diterapkan **dua kali** (satu per node, ganti `__NODE__`).

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: image-gc-prune-__NODE__
  namespace: kube-system
spec:
  schedule: "0 3 * * 0"            # Minggu 03:00
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      ttlSecondsAfterFinished: 3600
      template:
        spec:
          restartPolicy: Never
          nodeSelector:
            kubernetes.io/hostname: __NODE__   # vps-29 atau vps-38
          containers:
          - name: prune
            # Image harus mengandung binary crictl. Opsi: build image kecil dari
            # rancher/k3s:v1.36.5-k3s1 (ambil /usr/local/bin/crictl saja), atau
            # pakai langsung rancher/k3s:v1.36.5-k3s1 (lebih besar, ~ratusan MB).
            image: rancher/k3s:v1.36.5-k3s1
            command: ["crictl", "rmi", "--prune"]
            env:
            - name: CONTAINER_RUNTIME_ENDPOINT
              value: "unix:///run/k3s/containerd/containerd.sock"
            securityContext:
              runAsUser: 0
            volumeMounts:
            - name: containerd-sock
              mountPath: /run/k3s/containerd/containerd.sock
            resources:
              requests: {cpu: "100m", memory: "128Mi"}
              limits: {cpu: "500m", memory: "256Mi"}
          volumes:
          - name: containerd-sock
            hostPath:
              path: /run/k3s/containerd/containerd.sock
              type: Socket
```

**Risiko (baca sebelum menyetujui):**

1. Mount socket containerd + jalan sebagai root = **kontrol penuh atas runtime node** —
   setara root di node itu. Job ini bisa membuat/menghapus container apa pun bila
   perintahnya diubah. Hanya jalankan manifest dari sumber tepercaya.
2. Salah perintah (mis. `crictl rmi --all` alih-alih `--prune`) akan menghapus SEMUA image
   termasuk yang dipakai pod — pod tidak bisa restart sampai image di-pull ulang.
3. Jangan jadwalkan bersamaan dengan Job render: prune memang melindungi image yang sedang
   dipakai, tapi bentrok jadwal tetap dihindari (jadwal di atas jam 03:00 Minggu, render
   biasanya siang).

## 6. Ringkasan

- **Klaster sehat.** Tidak ada tekanan disk/memori di level kubelet, tidak ada pod gagal.
- **Satu-satunya penumpukan nyata: image store 45 GB di vps-29.** Sudah dibersihkan yang
  bisa dibersihkan lewat API (Job/pod completed + history CronJob). Sisanya butuh 2 langkah
  dari pemilik server: prune satu kali (§3A) + flag kubelet GC (§3B).
- **Memori vps-29 75–76%** — penyebabnya Rancher (2,3 GB, tanpa limit). Jalan keluar
  terdekat: render pindah ke vps-38 (§4A) + rencana tambah RAM tetap jalan. Batasi/hapus
  Rancher hanya bila diputuskan (§4B/§4C).
- **Jangan hapus Job `helm-install-*`** — dibuat ulang sendiri oleh controller.
