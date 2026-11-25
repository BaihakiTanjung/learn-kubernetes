# 01 — Menambah Worker Node ke Klaster k3s

Panduan langkah-demi-langkah menambahkan satu worker node ke klaster k3s yang sudah berjalan. Ditulis dari kasus nyata: **vps-38 bergabung ke klaster vps-29 pada 26 November 2026** — semua nilai, output, dan troubleshooting di bawah ini adalah yang benar-benar terjadi saat itu.

## Gambaran awal

| | Server (sudah ada) | Node baru |
| --- | --- | --- |
| Hostname | `vps-29` | `vps-38` |
| Peran | control-plane + etcd | worker (agent-only) |
| k3s | v1.36.5+k3s1 | v1.36.5+k3s1 |
| OS | Ubuntu 26.04.1 LTS | Ubuntu 26.04.1 LTS |
| Internal-IP | 10.20.0.18 | 10.20.0.6 |
| IP publik | 103.77.78.114 | 103.77.78.112 |
| podCIDR | 10.42.0.0/24 | 10.42.1.0/24 |

Arsitektur akhirnya didokumentasikan di [Architecture.md](../Architecture.md).

## Prasyarat

- Klaster k3s sudah berjalan di server (di sini: `vps-29`, single-node).
- VPS baru dengan OS Linux 64-bit — samakan dengan server, di sini Ubuntu 26.04.
  Untuk beban render, patokan minimal: **4 CPU / 8 GB RAM / 30 GB disk bebas**.
- Node baru harus bisa mencapai API server k3s di port **6443**.
- **Versi k3s agent harus sama persis dengan server** (`v1.36.5+k3s1` di kasus ini). Jangan pakai `latest` — versi yang berbeda bisa gagal join atau berperilaku aneh.

## Langkah 0 — Siapkan VPS baru

Buat VPS baru (di kasus ini: `vps-38`, user `rlordz121`, Ubuntu 26.04.1 LTS). Pastikan kamu bisa SSH ke sana dan menjalankan `sudo`.

## Langkah 1 — Tes konektivitas privat dari VPS baru

Dari VPS baru, cek apakah IP privat server terjangkau:

```bash
ping -c 2 10.20.0.18
```

Hasil kami:

```text
PING 10.20.0.18 (10.20.0.18) 56(84) bytes of data.
64 bytes from 10.20.0.18: icmp_seq=1 ttl=63 time=0.521 ms
64 bytes from 10.20.0.18: icmp_seq=2 ttl=63 time=0.575 ms

--- 10.20.0.18 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1024ms
rtt min/avg/max/mdev = 0.521/0.548/0.575/0.027 ms
```

**0% loss, ~0,5 ms = kedua VPS satu VPC.** Artinya join bisa lewat IP privat `https://10.20.0.18:6443` — ini jalur paling kecil risikonya, tanpa perlu Tailscale dan tanpa cek SAN sertifikat.

Kalau ping gagal, node baru tidak satu jaringan privat dengan server. Pilih salah satu jalur di [Alamat join alternatif](#alamat-join-alternatif) sebelum lanjut.

## Langkah 2 — Ambil node-token di server

Token join k3s (`node-token`) **hanya tersimpan sebagai file di server**, bukan Secret/ConfigMap — jadi token **tidak tersedia lewat API Kubernetes**. Ambil di `vps-29`:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

> ⚠️ **Jangan pernah tempel nilai token di chat, dan jangan pernah commit ke repo.**
> Token diisi langsung di mesin baru pada langkah 3, atau disampaikan lewat jalur kredensial yang aman bila memang harus diteruskan.

## Langkah 3 — Install agent di VPS baru

Jalankan **di VPS baru**. `K3S_URL` yang terisi membuat installer memasang **agent saja** — bukan server — jadi node baru tidak akan pernah menjadi control-plane atau etcd:

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_VERSION=v1.36.5+k3s1 \
  K3S_URL=https://10.20.0.18:6443 \
  K3S_TOKEN=<isi-node-token> \
  sh -
```

Ganti `<isi-node-token>` dengan nilai dari langkah 2 (diketik langsung di mesin baru). `INSTALL_K3S_VERSION` harus sama persis dengan versi server.

Lalu cek service-nya jalan:

```bash
sudo systemctl status k3s-agent --no-pager
sudo journalctl -u k3s-agent -n 50 --no-pager
```

Output yang sehat: service `active (running)`, log menunjukkan flannel mendapat subnet lease dari server.

## Langkah 4 — Troubleshooting: error `iptables-restore does not exist`

Saat pertama start, log kami berisi error ini:

```text
E1125 21:11:33.139563   28072 iptables.go:380] Failed to bootstrap IPTables: failed to apply partial iptables-restore unable to run iptables-restore... does not exist
```

**Penyebab:** paket `iptables` belum terpasang di VPS baru (image Ubuntu minimal kadang tidak menyertakannya). Kalau dibiarkan, networking pod di node ini bisa bermasalah.

**Perbaikan** (di VPS baru):

```bash
sudo apt-get update && sudo apt-get install -y iptables
sudo systemctl restart k3s-agent
```

## Langkah 5 — Verifikasi log setelah restart

```bash
sudo journalctl -u k3s-agent -n 30 --no-pager
```

Tanda-tanda join berhasil:

```text
I1125 21:12:48.842824   28072 kubelet_node_status.go:75] "Attempting to register node" node="vps-38"
I1125 21:12:48.884818   28072 kubelet_node_status.go:125] "Node was previously registered" node="vps-38"
I1125 21:12:48.885028   28072 kubelet_node_status.go:78] "Successfully registered node" node="vps-38"
I1125 21:12:48.885101   28072 kuberuntime_manager.go:2172] "Updating runtime config through cri with podcidr" CIDR="10.42.1.0/24"
...
time="2026-11-25T21:12:49Z" level=info msg="k3s agent is up and running"
```

Yang perlu dipastikan:

- Error `Failed to bootstrap IPTables` **sudah hilang**.
- Ada baris `Successfully registered node` dengan nama node baru.
- Node mendapat podCIDR sendiri (`10.42.1.0/24` — server memakai `10.42.0.0/24`).
- Baris terakhir: `k3s agent is up and running`.

Warning awal start seperti `PLEG is not healthy` atau `checkpoint is not found` adalah normal — keduanya hilang sendiri begitu kubelet stabil.

## Langkah 6 — Verifikasi dari server

Dari `vps-29`:

```bash
sudo /usr/local/bin/kubectl get nodes -o wide
```

Hasil kami:

```text
NAME     STATUS   ROLES           AGE   VERSION        INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION             CONTAINER-RUNTIME
vps-29   Ready    control-plane   24d   v1.36.5+k3s1   10.20.0.18    <none>        Ubuntu 26.04.1 LTS   7.0.0-42-generic (amd64)   containerd://2.3.4-k3s1.36
vps-38   Ready    <none>          98s   v1.36.5+k3s1   10.20.0.6     <none>        Ubuntu 26.04.1 LTS   7.0.0-34-generic (amd64)   containerd://2.3.4-k3s1.36
```

Node baru harus muncul dengan `STATUS Ready` dan `VERSION` yang sama dengan server. `ROLES <none>` saat ini normal — label peran dipasang di langkah 7.

## Langkah 7 — Beri label peran

Masih dari `vps-29`, beri label supaya jelas perannya dan supaya Job bisa menarget node ini dengan `nodeSelector`:

```bash
sudo /usr/local/bin/kubectl label node vps-38 node-role.kubernetes.io/worker=true workload=render --overwrite
```

Setelah label dipasang, `kubectl get nodes` menampilkan `ROLES worker` untuk `vps-38`.

> Catatan: kami sengaja memakai label saja, tanpa taint — pod yang sudah ada tetap boleh tersebar ke node baru bila diinginkan, dan Job render menargetnya lewat `nodeSelector: { workload: render }`.

## Langkah 8 — Uji penjadwalan dengan pod tes

Buat satu pod tes yang dipaksa jalan di node baru, cek, lalu hapus:

```bash
sudo /usr/local/bin/kubectl run join-test --image=nginx:alpine --overrides='{"spec":{"nodeSelector":{"workload":"render"}}}' --restart=Never
sudo /usr/local/bin/kubectl get pod join-test -o wide
sudo /usr/local/bin/kubectl delete pod join-test
```

Pod `join-test` harus muncul dengan `STATUS Running` dan kolom `NODE` berisi `vps-38`. Kalau itu terpenuhi, **worker sudah bisa menerima workload — setup selesai**.

Verifikasi akhir kami (26 Nov 2026 ~01:14): `vps-38` tetap `Ready`, `ROLES worker`, label `workload=render` dan `node-role.kubernetes.io/worker=true` terpasang.

## Alamat join alternatif

Kalau node baru **tidak** satu VPC dengan server, ada dua jalur lain. Keduanya mensyaratkan alamat server tercantum di SAN (Subject Alternative Name) sertifikat server — sertifikat k3s default hanya mencakup IP node server (`10.20.0.18`), jadi IP Tailscale/publik belum tentu tercakup.

**Cek dulu dari mesin baru** (ganti `<alamat-server>` dengan alamat yang mau dipakai):

```bash
openssl s_client -connect <alamat-server>:6443 -showcerts </dev/null 2>/dev/null \
  | openssl x509 -noout -text | grep -A 2 "Subject Alternative Name"
```

Alamat yang dipakai di `K3S_URL` **harus** muncul di daftar SAN.

| Jalur | `K3S_URL` | Syarat |
| --- | --- | --- |
| Jaringan privat provider (disarankan) | `https://10.20.0.18:6443` | Node baru bisa mencapai 10.20.0.18 |
| Tailscale | `https://100.123.155.8:6443` | `tailscale up` dulu di node baru, `ping 100.123.155.8` berhasil, alamat lolos cek SAN |
| Internet (IP publik) | `https://103.77.78.114:6443` | Alamat lolos cek SAN + firewall mengizinkan 6443 dari IP node baru |

Kalau alamat tidak muncul di SAN, **jangan pakai `--insecure-skip-tls-verify`**. Yang benar: tambah `--tls-san <alamat>` di config server (`/etc/rancher/k3s/config.yaml`), lalu restart k3s server — langkah ini butuh akses server langsung.

## Yang TIDAK dilakukan

- Tidak ada perubahan config di server selama proses join (tidak ada label/taint/config baru di `vps-29` sebelum langkah 7).
- Node-token tidak pernah diambil lewat jalur lain selain `sudo cat` langsung di server.
- Workload yang sedang berjalan tidak diganggu — semua pengecekan awal hanya `kubectl` read-only.
