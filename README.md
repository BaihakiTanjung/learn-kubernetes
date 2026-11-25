# learn-kubernetes

Dokumentasi belajar Kubernetes dari pengalaman nyata mengelola klaster k3s sendiri — ditulis berdasarkan langkah-langkah yang benar-benar dijalankan, bukan teori.

Klaster yang didokumentasikan di sini: **k3s v1.36.5+k3s1**, 2 node.

| Node | Peran | IP privat | IP publik |
| --- | --- | --- | --- |
| `vps-29` | Control-plane + etcd | 10.20.0.18 | 103.77.78.114 |
| `vps-38` | Worker | 10.20.0.6 | 103.77.78.112 |

## Isi repo

- [Architecture.md](Architecture.md) — arsitektur klaster hasil akhir: node, jaringan, namespace, dan keputusan desain.
- [docs/01-tambah-worker-node.md](docs/01-tambah-worker-node.md) — panduan langkah-demi-langkah menambahkan worker node ke klaster k3s (kasus nyata: vps-38 bergabung ke vps-29, 26 November 2026), lengkap dengan troubleshooting yang ditemui.
- [docs/02-garbage-collection-optimasi.md](docs/02-garbage-collection-optimasi.md) — garbage collection + optimasi klaster (kasus nyata: assessment 26 November 2026) — apa yang sudah dibersihkan lewat kubectl, apa yang menunggu tangan pemilik server, dan keputusan yang masih menunggu.

## Aturan keamanan

- **Jangan pernah commit node-token k3s** (`/var/lib/rancher/k3s/server/node-token`) atau kredensial apa pun ke repo ini. Token diambil langsung di server dan dipakai langsung di mesin baru, tidak lewat chat dan tidak lewat repo.
- Semua perintah di panduan ini memakai placeholder seperti `<isi-node-token>` untuk nilai sensitif.
