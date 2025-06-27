[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/V7fOtAk7)
|    NRP     |      Name      |
| :--------: | :------------: |
| 5025241017 | Mitra Partogi |
| 5025221036 | Muhammad Quthbi Danish Abqori |
| 5025241033 | Ferdian Ardra Hafizhan |

# Praktikum Modul 4 _(Module 4 Lab Work)_

</div>

### Daftar Soal _(Task List)_

- [Task 1 - FUSecure](/task-1/)

- [Task 2 - LawakFS++](/task-2/)

- [Task 3 - Drama Troll](/task-3/)

- [Task 4 - LilHabOS](/task-4/)

### Laporan Resmi Praktikum Modul 4 _(Module 4 Lab Work Report)_

Tulis laporan resmi di sini!

_Write your lab work report here!_

<h1>Task 1 - FUSecure</h1>

Langkah-langkah yang dilakukan dalam mengerjakan problem ini:
1. Instalasi dependensi (`libfuse3-dev`).
2. Persiapan user (`yuadi`, `irwandi`) dan direktori sumber dengan permission sesuai.
3. Membuat file uji di underlying source directory.
4. Struktur dan penjelasan `fusecure.c`.
5. Compile.
6. Mount dengan opsi `allow_other`.
7. Pengujian skenario baca public/private.

## 1. Instalasi Dependensi

1. **Update package list**:

   ```bash
   sudo apt update
   ```

   - Memperbarui daftar paket dari repositori.

2. **Install libfuse3-dev dan pkg-config**:

   ```bash
   sudo apt install -y libfuse3-dev pkg-config
   ```

   - `libfuse3-dev`: header dan library untuk mengembangkan filesystem FUSE versi 3.
   - `pkg-config`: alat untuk mengambil flag compile/link library (dipakai untuk `fuse3`).

3. **Verifikasi pemasangan**:

   ```bash
   pkg-config fuse3 --cflags
   pkg-config fuse3 --libs
   ```

   - `pkg-config fuse3 --cflags` mengeluarkan opsi compiler (misal `-I/usr/include/fuse3`).
   - `pkg-config fuse3 --libs` mengeluarkan opsi linker (misal `-lfuse3 -pthread`).
   - Jika tidak muncul, periksa pemasangan `libfuse3-dev`.

## 2. Persiapan User dan Direktori

### 2.1. Membuat Linux User

- `sudo useradd -m yuadi`
- `sudo passwd yuadi`
  - Atur password untuk user `yuadi`.
- Ulangi untuk `irwandi`:
  ```bash
  sudo useradd -m irwandi
  sudo passwd irwandi
  ```

### 2.2. Membuat Source Directory dan Subdirektori

- `sudo mkdir -p /home/shared_files/public`
- `sudo mkdir -p /home/shared_files/private_yuadi`
- `sudo mkdir -p /home/shared_files/private_irwandi`

### 2.3. Mengatur Kepemilikan dan Permission Underlying

- **public**:

  ```bash
  sudo chown root:root /home/shared_files/public
  sudo chmod 755 /home/shared_files/public
  ```

  - `chown root:root`: pemilik root, group root.
  - `chmod 755`: owner baca/tulis/execute, group dan others baca/execute. Direktori perlu execute untuk bisa masuk.
  - Artinya, underlying filesystem mengijinkan siapa saja baca isi public.

- **private\_yuadi**:

  ```bash
  sudo chown yuadi:yuadi /home/shared_files/private_yuadi
  sudo chmod 700 /home/shared_files/private_yuadi
  ```

  - `chown yuadi:yuadi`: pemilik direksi adalah user `yuadi`.
  - `chmod 700`: hanya owner dapat baca/tulis/execute (execute untuk masuk direktori).
  - Underlying, hanya `yuadi` yang dapat baca di luar konteks FUSE; tetapi FUSE dijalankan sebagai root dapat membaca underlying, namun logika FUSE akan membatasi.

- **private\_irwandi**:

  ```bash
  sudo chown irwandi:irwandi /home/shared_files/private_irwandi
  sudo chmod 700 /home/shared_files/private_irwandi
  ```

- **Penjelasan**: Pengaturan permission underlying membantu keamanan. Meski FUSE mengontrol, underlying permission tidak mengizinkan user lain di luar FUSE untuk langsung mengakses folder private.

## 3. Membuat File Uji di Source Directory

> Karena mountpoint FUSE read-only, membuat file uji dilakukan di underlying source directory.

### 3.1. File di `public`

- **Perintah**:
  ```bash
  sudo bash -c 'echo "Isi materi publik untuk tes" > /home/shared_files/public/test_public.txt'
  sudo chmod 644 /home/shared_files/public/test_public.txt
  ```
  - `sudo bash -c`: menjalankan perintah sebagai root sehingga memiliki izin tulis.
  - `chmod 644`: owner baca/tulis, group/others baca.
  - Setelah ini, file bisa dibaca lewat FUSE oleh siapa saja.

### 3.2. File di `private_yuadi`

- **Perintah**:
  ```bash
  sudo -u yuadi bash -c 'echo "Isi private Yuadi" > /home/shared_files/private_yuadi/test_yuadi.txt'
  sudo chmod 600 /home/shared_files/private_yuadi/test_yuadi.txt
  ```
  - `sudo -u yuadi`: jalankan sebagai user `yuadi` sehingga file otomatis owner `yuadi`.
  - `chmod 600`: hanya owner (yuadi) baca/tulis.
  - FUSE akan mengizinkan baca oleh `yuadi`, menolak untuk user lain.

### 3.3. File di `private_irwandi`

- **Perintah**:
  ```bash
  sudo -u irwandi bash -c 'echo "Isi private Irwandi" > /home/shared_files/private_irwandi/test_irwandi.txt'
  sudo chmod 600 /home/shared_files/private_irwandi/test_irwandi.txt
  ```

## 4. Struktur `fusecure.c` dan Penjelasan

Berikut ringkasan fungsi-fungsi utama dan alur kerja `fusecure.c`. Implementasi lengkap dapat disertakan sebagai lampiran atau di bagian kode.

### 4.1. Header dan Inisialisasi

```c
#define FUSE_USE_VERSION 35
#include <fuse3/fuse.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <dirent.h>
#include <limits.h>
#include <pwd.h>

static char *rootdir = NULL;
```

- `FUSE_USE_VERSION 35`: menentukan API FUSE versi 3.5.
- Include header FUSE3 dan library sistem.
- `rootdir`: menyimpan path absolut source directory.

### 4.2. Fungsi `fullpath`

```c
static void fullpath(char outpath[PATH_MAX], const char *path) {
    snprintf(outpath, PATH_MAX, "%s%s", rootdir, path);
}
```

- Menggabungkan `rootdir` (misal `/home/shared_files`) dengan `path` dari FUSE (misal `/public/test.txt`) menjadi path di underlying: `/home/shared_files/public/test.txt`.
- `snprintf` memastikan tidak overflow buffer.

### 4.3. Deteksi Private Path: `path_is_private`

```c
static int path_is_private(const char *path, char *username, size_t maxlen) {
    const char prefix[] = "/private_";
    size_t plen = strlen(prefix);
    if (strncmp(path, prefix, plen) != 0)
        return 0;
    const char *p = path + plen;
    size_t i = 0;
    while (p[i] && p[i] != '/') {
        if (i + 1 >= maxlen) break;
        i++;
    }
    if (i == 0 || i >= maxlen)
        return 0;
    strncpy(username, p, i);
    username[i] = '\0';
    return 1;
}
```

- Memeriksa apakah `path` diawali `/private_`.
- Jika ya, mengekstrak nama user antara prefix dan slash berikutnya, menyimpan di `username`.
- Contoh: `path = "/private_yuadi/file.txt"` → `username = "yuadi"`.
- `maxlen` mencegah buffer overflow.

### 4.4. Pengecekan Izin Akses Private: `check_private_allowed`

```c
static int check_private_allowed(const char *path) {
    char uname[256];
    if (!path_is_private(path, uname, sizeof(uname)))
        return 1; // bukan private, izinkan lanjut
    struct passwd *pw = getpwnam(uname);
    if (!pw) return 0; // user tidak ada
    uid_t allowed_uid = pw->pw_uid;
    struct fuse_context *fc = fuse_get_context();
    if (!fc) return 0;
    if (fc->uid == allowed_uid) return 1;
    return 0;
}
```

- Jika path berada di bawah `/private_<user>`, ambil `<user>` dan cari UID-nya lewat `getpwnam`.
- Dapatkan UID pemanggil melalui `fuse_get_context()->uid`.
- Jika sama, akses diizinkan; jika tidak, tolak dengan EACCES.
- Jika path bukan private, kembalikan 1: izinkan pengecekan selanjutnya.

### 4.5. Callback FUSE untuk operasi baca dan metadata

- **getattr**:

  ```c
  static int secure_getattr(const char *path, struct stat *stbuf, struct fuse_file_info *fi) {
      (void) fi;
      if (!check_private_allowed(path))
          return -EACCES;
      char fpath[PATH_MAX];
      fullpath(fpath, path);
      int res = lstat(fpath, stbuf);
      if (res == -1)
          return -errno;
      return 0;
  }
  ```

  - Mengecek izin private.
  - Memanggil `lstat` pada underlying file untuk mengisi `stbuf`.
  - Jika file tidak ada, mengembalikan `-ENOENT`, dsb.

- **access**:

  ```c
  static int secure_access(const char *path, int mask) {
      if (!check_private_allowed(path))
          return -EACCES;
      if (mask & W_OK)
          return -EROFS;
      char fpath[PATH_MAX]; fullpath(fpath, path);
      int res = access(fpath, mask & ~W_OK);
      if (res == -1)
          return -errno;
      return 0;
  }
  ```

  - Menolak write (`W_OK`) dengan `-EROFS` (Read-only filesystem).
  - Untuk read/execute/existence, delegasikan ke `access` underlying.
  - Jika underlying menolak (misal file tidak readable oleh user underlying?), `-errno` dikembalikan.

- **open**:

  ```c
  static int secure_open(const char *path, struct fuse_file_info *fi) {
      if (!check_private_allowed(path))
          return -EACCES;
      int flags = fi->flags;
      if ((flags & O_ACCMODE) != O_RDONLY)
          return -EACCES;
      char fpath[PATH_MAX]; fullpath(fpath, path);
      int fd = open(fpath, O_RDONLY);
      if (fd == -1)
          return -errno;
      fi->fh = fd;
      return 0;
  }
  ```

  - Pastikan hanya O\_RDONLY.
  - Buka underlying file dengan `open(..., O_RDONLY)`, simpan file descriptor di `fi->fh`.

- **read**:

  ```c
  static int secure_read(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
      (void) path;
      int res = pread(fi->fh, buf, size, offset);
      if (res == -1)
          res = -errno;
      return res;
  }
  ```

  - Membaca dari file descriptor yang dibuka di `open`.

- **release**:

  ```c
  static int secure_release(const char *path, struct fuse_file_info *fi) {
      (void) path;
      close(fi->fh);
      return 0;
  }
  ```

  - Menutup FD.

- **readdir**:

  ```c
  static int secure_readdir(const char *path, void *buf, fuse_fill_dir_t filler,
                            off_t offset, struct fuse_file_info *fi, enum fuse_readdir_flags flags) {
      (void) offset; (void) fi; (void) flags;
      if (!check_private_allowed(path))
          return -EACCES;
      char fpath[PATH_MAX]; fullpath(fpath, path);
      DIR *dp = opendir(fpath);
      if (!dp) return -errno;
      struct dirent *de;
      filler(buf, ".", NULL, 0, 0);
      filler(buf, "..", NULL, 0, 0);
      while ((de = readdir(dp)) != NULL) {
          filler(buf, de->d_name, NULL, 0, 0);
      }
      closedir(dp);
      return 0;
  }
  ```

  - Cek izin private.
  - `opendir` pada underlying direktori.
  - Tambahkan entri `.` dan `..`, lalu entri lainnya.
  - Jika direktori private dan user tidak sesuai, gagal di awal dengan `-EACCES`.

- **statfs**:

  ```c
  static int secure_statfs(const char *path, struct statvfs *stbuf) {
      char fpath[PATH_MAX]; fullpath(fpath, path);
      int res = statvfs(fpath, stbuf);
      if (res == -1)
          return -errno;
      return 0;
  }
  ```

  - Menampilkan statistik filesystem underlying.

- **readlink**:

  ```c
  static int secure_readlink(const char *path, char *buf, size_t size) {
      if (!check_private_allowed(path))
          return -EACCES;
      char fpath[PATH_MAX]; fullpath(fpath, path);
      int res = readlink(fpath, buf, size - 1);
      if (res == -1) return -errno;
      buf[res] = '\0';
      return 0;
  }
  ```

  - Membaca link target.
  - Jika target di luar rootdir, bisa ditolak jika diinginkan.

### 4.6. Stub Operasi Write / Modify

Semua operasi yang berpotensi memodifikasi filesystem (write, mkdir, unlink, rmdir, rename, chmod, utimens, chown, create, mknod, truncate, dsb.) diimplementasikan sebagai stub yang mengembalikan `-EROFS`:

```c
static int secure_mkdir(const char *path, mode_t mode) { return -EROFS; }
static int secure_unlink(const char *path) { return -EROFS; }
// dll.
```

- `-EROFS` artinya "Read-only file system". Memastikan operasi semacam `mkdir /mnt/secure_fs/newdir` gagal.

### 4.7. Struktur `fuse_operations`

```c
static const struct fuse_operations secure_ops = {
    .getattr    = secure_getattr,
    .access     = secure_access,
    .open       = secure_open,
    .read       = secure_read,
    .release    = secure_release,
    .readdir    = secure_readdir,
    .statfs     = secure_statfs,
    .readlink   = secure_readlink,
    .mkdir      = secure_mkdir,
    .unlink     = secure_unlink,
    .rmdir      = secure_rmdir,
    .rename     = secure_rename,
    .mknod      = secure_mknod,
    .write      = secure_write,
    .chmod      = secure_chmod,
    .utimens    = secure_utimens,
    // Tambahkan stub lain bila diperlukan
};
```

- Menentukan callback untuk setiap operasi filesystem yang didukung.
- Operasi write diarahkan ke stub yang menolak.

### 4.8. Fungsi `main`

```c
int main(int argc, char *argv[]) {
    if (argc < 3) {
        fprintf(stderr, "Usage: %s <source_dir> <mountpoint> [fuse options]\n", argv[0]);
        exit(1);
    }
    char absroot[PATH_MAX];
    if (!realpath(argv[1], absroot)) {
        perror("realpath source_dir");
        exit(1);
    }
    rootdir = strdup(absroot);
    for (int i = 1; i < argc - 1; i++) {
        argv[i] = argv[i+1];
    }
    argv[argc-1] = NULL;
    argc--;
    return fuse_main(argc, argv, &secure_ops, NULL);
}
```

- Memeriksa argumen: minimal dua argumen: `source_dir` dan `mountpoint`.
- `realpath` memastikan source directory dalam path absolut.
- Menyimpan `rootdir`.
- Memodifikasi `argv` agar FUSE library menerima parameter mountpoint dan opsi lainnya.
- Memanggil `fuse_main` untuk menjalankan loop FUSE.

## 5. Compile dan Mount

### 5.1. Compile

- **Perintah**:
  ```bash
  gcc `pkg-config fuse3 --cflags` fusecure.c -o fusecure `pkg-config fuse3 --libs`
  ```
  - `pkg-config fuse3 --cflags`: menambahkan opsi compiler (include path untuk header FUSE3).
  - `fusecure.c`: file source.
  - `-o fusecure`: nama binary output.
  - `pkg-config fuse3 --libs`: menambahkan opsi linker (library FUSE3 dan dependensinya).
- **Penjelasan**: Jika urutan flags link salah (misal `--libs` sebelum source), dapat muncul error undefined reference ke fungsi FUSE.
- Setelah compile, muncul executable `fusecure`.

### 5.2. Pastikan `/etc/fuse.conf`

- **Perintah**:
  ```bash
  grep "user_allow_other" /etc/fuse.conf
  ```
- Jika tidak ada atau dikomentari (`#user_allow_other`), edit:
  ```bash
  sudo nano /etc/fuse.conf
  ```
  - Hilangkan `#` pada `user_allow_other`.
  - Simpan.
- Opsi `allow_other` diperlukan agar user selain yang mount (sering root) dapat mengakses mountpoint.
![Screenshot 2025-06-18 010113](https://github.com/user-attachments/assets/2531a53b-449c-4f0f-85e4-4f861b3c3f05)


### 5.3. Buat mount point

- **Perintah**:
  ```bash
  sudo mkdir -p /mnt/secure_fs
  ```
  - Membuat direktori mount point.

### 5.4. Mount FUSE

- **Perintah**:
  ```bash
  sudo ./fusecure /home/shared_files /mnt/secure_fs -o allow_other
  ```
  - `sudo`: menjalankan sebagai root (agar dapat mount dan mengizinkan akses user lain).
  - `./fusecure`: executable FUSE.
  - `/home/shared_files`: source\_dir (rootdir underlying).
  - `/mnt/secure_fs`: mount point.
  - `-o allow_other`: mengizinkan user lain selain pemanggil mount untuk mengakses.

## 6. Pengujian

Setelah mount berhasil, lakukan pengujian skenario.

### 6.1. Sebelum pengujian: Pastikan file uji sudah ada di underlying

- Public: `/home/shared_files/public/test_public.txt`.
- Private Yuadi: `/home/shared_files/private_yuadi/test_yuadi.txt`.
- Private Irwandi: `/home/shared_files/private_irwandi/test_irwandi.txt`.

### 6.2. Sebagai user `yuadi`

```bash
su - yuadi
cat /mnt/secure_fs/public/test_public.txt
cat /mnt/secure_fs/private_yuadi/test_yuadi.txt
cat /mnt/secure_fs/private_irwandi/test_irwandi.txt
touch /mnt/secure_fs/public/oke.txt
```

- `cat`: menampilkan isi file.
- `touch`: membuat file baru, pada read-only FS harusnya gagal.

![Screenshot 2025-06-18 151452](https://github.com/user-attachments/assets/6852871d-f5b2-4903-b8ce-d9b196290252)


### 6.3. Sebagai user `irwandi`

```bash
su - irwandi
cat /mnt/secure_fs/public/test_public.txt
cat /mnt/secure_fs/private_irwandi/test_irwandi.txt
cat /mnt/secure_fs/private_yuadi/test_yuadi.txt
mkdir /mnt/secure_fs/coba
```

- `mkdir`: akan gagal.
  
![Screenshot 2025-06-18 151644](https://github.com/user-attachments/assets/e06d6c92-eed9-4f34-ba6c-e2804aab51da)


### 6.4. Sebagai user lain (misal sebagai lixyon)

```bash
cat /mnt/secure_fs/public/test_public.txt
ls /mnt/secure_fs/private_yuadi
cat /mnt/secure_fs/private_irwandi/test_irwandi.txt
```

- Hanya akses public diizinkan.
  
![Screenshot 2025-06-18 151317](https://github.com/user-attachments/assets/7a990b14-4496-4e6e-87f9-2cb91cf0db70)

### 6.5. Memeriksa error code dan pesan

- Jika `cat` pada private non-owner gagal, muncul `Permission denied`.
- Jika operasi write (touch/mkdir) gagal, muncul `Read-only file system`.
- Jika `ls /mnt/secure_fs/private_yuadi` gagal, muncul `Permission denied`.

## 7. Unmount FUSE

   ```bash
   sudo fusermount3 -u /mnt/secure_fs
   ```
---

*Lampiran: fusecure.c*

```c
#define FUSE_USE_VERSION 35
#include <fuse3/fuse.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <dirent.h>
#include <limits.h>
#include <pwd.h>

static char *rootdir = NULL;

static void fullpath(char outpath[PATH_MAX], const char *path) {
    snprintf(outpath, PATH_MAX, "%s%s", rootdir, path);
}

static int path_is_private(const char *path, char *username, size_t maxlen) {
    const char prefix[] = "/private_";
    size_t plen = strlen(prefix);
    if (strncmp(path, prefix, plen) != 0)
        return 0;
    const char *p = path + plen;
    size_t i = 0;
    while (p[i] && p[i] != '/') {
        if (i + 1 >= maxlen) break;
        i++;
    }
    if (i == 0 || i >= maxlen)
        return 0;
    strncpy(username, p, i);
    username[i] = '\0';
    return 1;
}

static int check_private_allowed(const char *path) {
    char uname[256];
    if (!path_is_private(path, uname, sizeof(uname)))
        return 1;
    struct passwd *pw = getpwnam(uname);
    if (!pw) return 0;
    uid_t allowed_uid = pw->pw_uid;
    struct fuse_context *fc = fuse_get_context();
    if (!fc) return 0;
    if (fc->uid == allowed_uid) return 1;
    return 0;
}

static int secure_getattr(const char *path, struct stat *stbuf, struct fuse_file_info *fi) {
    (void) fi;
    if (!check_private_allowed(path))
        return -EACCES;
    char fpath[PATH_MAX]; fullpath(fpath, path);
    int res = lstat(fpath, stbuf);
    if (res == -1) return -errno;
    return 0;
}

static int secure_access(const char *path, int mask) {
    if (!check_private_allowed(path))
        return -EACCES;
    if (mask & W_OK)
        return -EROFS;
    char fpath[PATH_MAX]; fullpath(fpath, path);
    int res = access(fpath, mask & ~W_OK);
    if (res == -1) return -errno;
    return 0;
}

static int secure_open(const char *path, struct fuse_file_info *fi) {
    if (!check_private_allowed(path))
        return -EACCES;
    int flags = fi->flags;
    if ((flags & O_ACCMODE) != O_RDONLY)
        return -EACCES;
    char fpath[PATH_MAX]; fullpath(fpath, path);
    int fd = open(fpath, O_RDONLY);
    if (fd == -1) return -errno;
    fi->fh = fd;
    return 0;
}

static int secure_read(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
    (void) path;
    int res = pread(fi->fh, buf, size, offset);
    if (res == -1) res = -errno;
    return res;
}

static int secure_release(const char *path, struct fuse_file_info *fi) {
    (void) path;
    close(fi->fh);
    return 0;
}

static int secure_readdir(const char *path, void *buf, fuse_fill_dir_t filler,
                          off_t offset, struct fuse_file_info *fi, enum fuse_readdir_flags flags) {
    (void) offset; (void) fi; (void) flags;
    if (!check_private_allowed(path))
        return -EACCES;
    char fpath[PATH_MAX]; fullpath(fpath, path);
    DIR *dp = opendir(fpath);
    if (!dp) return -errno;
    struct dirent *de;
    filler(buf, ".", NULL, 0, 0);
    filler(buf, "..", NULL, 0, 0);
    while ((de = readdir(dp)) != NULL) {
        filler(buf, de->d_name, NULL, 0, 0);
    }
    closedir(dp);
    return 0;
}

static int secure_statfs(const char *path, struct statvfs *stbuf) {
    char fpath[PATH_MAX]; fullpath(fpath, path);
    int res = statvfs(fpath, stbuf);
    if (res == -1) return -errno;
    return 0;
}

static int secure_readlink(const char *path, char *buf, size_t size) {
    if (!check_private_allowed(path))
        return -EACCES;
    char fpath[PATH_MAX]; fullpath(fpath, path);
    int res = readlink(fpath, buf, size - 1);
    if (res == -1) return -errno;
    buf[res] = '\0';
    return 0;
}

static int secure_mkdir(const char *path, mode_t mode) { (void)path; (void)mode; return -EROFS; }
static int secure_unlink(const char *path) { (void)path; return -EROFS; }
static int secure_rmdir(const char *path) { (void)path; return -EROFS; }
static int secure_rename(const char *from, const char *to, unsigned int flags) { (void)from; (void)to; (void)flags; return -EROFS; }
static int secure_mknod(const char *path, mode_t mode, dev_t rdev) { (void)path; (void)mode; (void)rdev; return -EROFS; }
static int secure_write(const char *path, const char *buf, size_t size, off_t offset, struct fuse_file_info *fi) { (void)path; (void)buf; (void)size; (void)offset; (void)fi; return -EROFS; }
static int secure_chmod(const char *path, mode_t mode, struct fuse_file_info *fi) { (void)path; (void)mode; (void)fi; return -EROFS; }
static int secure_utimens(const char *path, const struct timespec tv[2], struct fuse_file_info *fi) { (void)path; (void)tv; (void)fi; return -EROFS; }

static const struct fuse_operations secure_ops = {
    .getattr    = secure_getattr,
    .access     = secure_access,
    .open       = secure_open,
    .read       = secure_read,
    .release    = secure_release,
    .readdir    = secure_readdir,
    .statfs     = secure_statfs,
    .readlink   = secure_readlink,
    .mkdir      = secure_mkdir,
    .unlink     = secure_unlink,
    .rmdir      = secure_rmdir,
    .rename     = secure_rename,
    .mknod      = secure_mknod,
    .write      = secure_write,
    .chmod      = secure_chmod,
    .utimens    = secure_utimens,
};

int main(int argc, char *argv[]) {
    if (argc < 3) {
        fprintf(stderr, "Usage: %s <source_dir> <mountpoint> [fuse options]\n", argv[0]);
        exit(1);
    }
    char absroot[PATH_MAX];
    if (!realpath(argv[1], absroot)) {
        perror("realpath source_dir"); exit(1);
    }
    rootdir = strdup(absroot);
    for (int i = 1; i < argc - 1; i++) {
        argv[i] = argv[i+1];
    }
    argv[argc-1] = NULL;
    argc--;
    return fuse_main(argc, argv, &secure_ops, NULL);
}
```

<h1>Task 2 - LawakFS</h1>

```c
#define FUSE_USE_VERSION 35

#include <fuse3/fuse.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <dirent.h>
#include <errno.h>
#include <sys/stat.h>
#include <time.h>
#include <stdlib.h>
#include <libgen.h>
#include <ctype.h>
#include <openssl/bio.h>
#include <openssl/evp.h>
#include <openssl/buffer.h>

#define MAX_FILENAME 256
#define MAX_PATH 4096
#define MAX_LAWAK_WORDS 100
#define MAX_WORD_LEN 32

char lawak_words[MAX_LAWAK_WORDS][MAX_WORD_LEN];
int lawak_count = 0;
char secret_basename[MAX_FILENAME] = "secret";
int access_start = 8;    // 08:00
int access_end = 18;     // 18:00

static const char *source_dir = "/home/ferdian-ardra/praktikum_linux/LawakFS++";


//soal d
#define LOG_PATH "/home/ferdian-ardra/praktikum_linux/modul4/sisop-modul-4-mitrapartogi13/task-2/var/log/lawakfs.log"
void log_action(const char *action, const char *path) {
    FILE *log_file = fopen(LOG_PATH, "a");
    if (!log_file) return;

    time_t t = time(NULL);
    struct tm tm_info;
    localtime_r(&t, &tm_info);

    char timestamp[20];
    strftime(timestamp, sizeof(timestamp), "%F %T", &tm_info);

    uid_t uid = getuid();

    fprintf(log_file, "[%s] [%d] [%s] [%s]\n", timestamp, uid, action, path);
    fclose(log_file);
}

// soal c
char* base64_encode(const unsigned char *input, size_t length, size_t *output_len) {
    BIO *bio, *b64;
    BUF_MEM *bufferPtr;

    b64 = BIO_new(BIO_f_base64());
    bio = BIO_new(BIO_s_mem());
    bio = BIO_push(b64, bio);

    BIO_set_flags(bio, BIO_FLAGS_BASE64_NO_NL);
    BIO_write(bio, input, length);
    BIO_flush(bio);
    BIO_get_mem_ptr(bio, &bufferPtr);
    
    char *encoded = malloc(bufferPtr->length + 1);
    if (!encoded) {
        BIO_free_all(bio);
        return NULL;
    }
    
    memcpy(encoded, bufferPtr->data, bufferPtr->length);
    encoded[bufferPtr->length] = '\0';
    *output_len = bufferPtr->length;
    
    BIO_free_all(bio);
    return encoded;
}

// soal c
static int is_text_file(const char *path) {
    int fd = open(path, O_RDONLY);
    if (fd == -1) return 0;
    
    char buf[1024];
    ssize_t bytes = read(fd, buf, sizeof(buf));
    close(fd);
    
    if (bytes <= 0) return 0;

    for (ssize_t i = 0; i < bytes; i++) {
        if (buf[i] == '\0') return 0;
    }
    return 1;
}

// soal e
void load_lawak_config(const char *config_path) {
    printf("[CONFIG] Loading configuration from: %s\n", config_path);
    
    FILE *fp = fopen(config_path, "r");
    if (!fp) {
        printf("[WARNING] Could not open config file %s: %s\n", 
               config_path, strerror(errno));
        printf("[CONFIG] Using default values\n");
        return;
    }

    char line[MAX_WORD_LEN * 2]; 
    while (fgets(line, sizeof(line), fp)) {
        line[strcspn(line, "\n")] = '\0';
        if (line[0] == '\0' || line[0] == '#') continue;

        char *key = strtok(line, "=");
        char *value = strtok(NULL, "=");
        
        if (!key || !value) continue;

        if (strcmp(key, "SECRET_FILE_BASENAME") == 0) {
            strncpy(secret_basename, value, MAX_FILENAME - 1);
            secret_basename[MAX_FILENAME - 1] = '\0';
            printf("[CONFIG] Set secret_basename: %s\n", secret_basename);
        } 
        else if (strcmp(key, "access_start") == 0) {
            access_start = atoi(value);
            printf("[CONFIG] Set access_start: %02d:00\n", access_start);
        }
        else if (strcmp(key, "access_end") == 0) {
            access_end = atoi(value);
            printf("[CONFIG] Set access_end: %02d:00\n", access_end);
        }
        else if (strcmp(key, "lawak_words") == 0) {
            char *word = strtok(value, ",");
            while (word && lawak_count < MAX_LAWAK_WORDS) {
                while (isspace(*word)) word++;
                char *end = word + strlen(word) - 1;
                while (end > word && isspace(*end)) end--;
                *(end + 1) = '\0';

                if (*word) {
                    strncpy(lawak_words[lawak_count], word, MAX_WORD_LEN - 1);
                    lawak_words[lawak_count][MAX_WORD_LEN - 1] = '\0';
                    printf("[CONFIG] Added lawak word: %s\n", lawak_words[lawak_count]);
                    lawak_count++;
                }
                word = strtok(NULL, ",");
            }
        }
    }
    fclose(fp);
}

// soal b
static int is_secret_file(const char *filename) {
    const char *base = strrchr(filename, '/');
    base = base ? base + 1 : filename;
    return strncmp(base, secret_basename, strlen(secret_basename)) == 0;
}

// soal b
static int is_accessible_time() {
    time_t now = time(NULL);
    struct tm *tm_now = localtime(&now);
    int hour = tm_now->tm_hour;
    return (hour >= access_start && hour < access_end);
}

// soal a
static int hidden_path(char *resolved_path, const char *virtual_path) {
    char virtual_name[MAX_PATH];
    snprintf(virtual_name, sizeof(virtual_name), "%s", virtual_path);
    const char *filename = virtual_name[0] == '/' ? virtual_name + 1 : virtual_name;

    DIR *dp = opendir(source_dir);
    if (!dp) return -errno;

    struct dirent *de;
    while ((de = readdir(dp)) != NULL) {
        if (de->d_name[0] == '.') continue;

        char name_only[MAX_FILENAME];
        strncpy(name_only, de->d_name, MAX_FILENAME - 1);
        name_only[MAX_FILENAME - 1] = '\0';

        char *dot = strrchr(name_only, '.');
        if (dot) *dot = '\0';

        if (strcmp(name_only, filename) == 0) {
            snprintf(resolved_path, MAX_PATH, "%s/%s", source_dir, de->d_name);
            closedir(dp);
            return 0;
        }
    }

    closedir(dp);
    return -ENOENT;
}

static int lawakfs_getattr(const char *path, struct stat *stbuf,
                           struct fuse_file_info *fi) {
    (void) fi;
    char full_path[MAX_PATH];

    if (strcmp(path, "/") == 0) {
        memset(stbuf, 0, sizeof(struct stat));
        stbuf->st_mode = __S_IFDIR | 0555;
        stbuf->st_nlink = 2;
        return 0;
    }

    // perubahan untuk nomer b
    if (is_secret_file(path) && !is_accessible_time()) 
        return -ENOENT;

    // soal a
    if (hidden_path(full_path, path) != 0)
        return -ENOENT;

    struct stat temp_stat;
    if (lstat(full_path, &temp_stat) == -1)
        return -errno;

    if (is_text_file(full_path)) {
        stbuf->st_size *= 4; 

    memcpy(stbuf, &temp_stat, sizeof(struct stat));

    stbuf->st_mode &= ~(S_IWUSR | S_IWGRP | S_IWOTH);
    return 0;
}

static int lawakfs_readdir(const char *path, void *buf, fuse_fill_dir_t filler,
                           off_t offset, struct fuse_file_info *fi,
                           enum fuse_readdir_flags flags) {
    (void) offset; (void) fi; (void) flags;

    if (strcmp(path, "/") != 0)
        return -ENOENT;

    DIR *dp = opendir(source_dir);
    if (!dp) return -errno;

    filler(buf, ".", NULL, 0, 0);
    filler(buf, "..", NULL, 0, 0);

    struct dirent *de;
    while ((de = readdir(dp)) != NULL) {
        if (de->d_name[0] == '.') continue;
    // soal b
    if (is_secret_file(de->d_name) && !is_accessible_time()) continue;

        

        char name_copy[MAX_FILENAME];
        strncpy(name_copy, de->d_name, MAX_FILENAME - 1);
        name_copy[MAX_FILENAME - 1] = '\0';

        // perubahan untuk nomor a
        char *dot = strrchr(name_copy, '.');
        if (dot) *dot = '\0';

        if (filler(buf, name_copy, NULL, 0, 0) != 0) {
            closedir(dp);
            return -ENOMEM;
        }
    }

    closedir(dp);
    return 0;
}

static int lawakfs_open(const char *path, struct fuse_file_info *fi) {
    char full_path[MAX_PATH];
    if (hidden_path(full_path, path) != 0)
    return -ENOENT;


    int fd = open(full_path, O_RDONLY);
    if (fd == -1) return -errno;

    close(fd);
    return 0;
}

static int lawakfs_read(const char *path, char *buf, size_t size, off_t offset,
                        struct fuse_file_info *fi) {
    (void) fi;

    char full_path[MAX_PATH];
    if (hidden_path(full_path, path) != 0)
        return -ENOENT;

    int fd = open(full_path, O_RDONLY);
    if (fd == -1) return -errno;

    struct stat st;
    fstat(fd, &st);
    char *content = malloc(st.st_size + 1);
    if (!content) {
        close(fd);
        return -ENOMEM;
    }

    ssize_t res = pread(fd, content, st.st_size, 0);
    if (res == -1) {
        free(content);
        close(fd);
        return -errno;
    }
    content[res] = '\0'; 

    char *output = NULL;
    size_t output_len = 0;

    if (is_text_file(full_path)) {
        size_t capacity = res * 4; 
        output = malloc(capacity);
        if (!output) {
            free(content);
            close(fd);
            return -ENOMEM;
        }

        size_t input_pos = 0, output_pos = 0;
        while (input_pos < res) {
            int word_matched = 0;

            for (int i = 0; i < lawak_count; ++i) {
                size_t word_len = strlen(lawak_words[i]);

                if (input_pos + word_len <= res &&
                    strncasecmp(&content[input_pos], lawak_words[i], word_len) == 0) {
                    
                    char prev = (input_pos == 0) ? ' ' : content[input_pos - 1];
                    char next = (input_pos + word_len >= res) ? ' ' : content[input_pos + word_len];

                    if (!isalnum(prev) && !isalnum(next)) {
                        memcpy(&output[output_pos], "lawak", 5);
                        output_pos += 5;
                        input_pos += word_len;
                        word_matched = 1;
                        break;
                    }
                }
            }

            if (!word_matched) {
                output[output_pos++] = content[input_pos++];
            }
        }

        
        if (output_pos > 0 && output[output_pos - 1] != '\n') {
            output[output_pos++] = '\n';
        }

        output[output_pos] = '\0';
        output_len = output_pos;
    } else {
        output = base64_encode((const unsigned char *)content, res, &output_len);
        if (!output) {
            free(content);
            close(fd);
            return -ENOMEM;
        }
    }

    size_t bytes_to_copy = 0;
    if (offset < output_len) {
        bytes_to_copy = (offset + size > output_len) ? output_len - offset : size;
        memcpy(buf, output + offset, bytes_to_copy);
    }

    free(content);
    free(output);
    close(fd);

    log_action("READ", path);
    return bytes_to_copy;
}


static int lawakfs_write(const char *path, const char *buf, size_t size, off_t offset,
                         struct fuse_file_info *fi) {
    return -EROFS;
}
static int lawakfs_truncate(const char *path, off_t size,
                            struct fuse_file_info *fi) {
    return -EROFS;
}
static int lawakfs_create(const char *path, mode_t mode,
                          struct fuse_file_info *fi) {
    return -EROFS;
}
static int lawakfs_unlink(const char *path) { return -EROFS; }
static int lawakfs_mkdir(const char *path, mode_t mode) { return -EROFS; }
static int lawakfs_rmdir(const char *path) { return -EROFS; }
static int lawakfs_rename(const char *from, const char *to, unsigned int flags) { return -EROFS; }
static int lawakfs_access(const char *path, int mask) {
    if (is_secret_file(path) && !is_accessible_time()) {
        return -ENOENT;
    }
    if (mask & W_OK){
         return -EROFS;
    }
    log_action("ACCESS", path);
    return 0;
}

static struct fuse_operations lawakfs_oper = {
    .getattr = lawakfs_getattr,
    .readdir = lawakfs_readdir,
    .read    = lawakfs_read,
    .open    = lawakfs_open,
    .access  = lawakfs_access,
    .write   = lawakfs_write,
    .truncate = lawakfs_truncate,
    .create = lawakfs_create,
    .unlink = lawakfs_unlink,
    .mkdir  = lawakfs_mkdir,
    .rmdir  = lawakfs_rmdir,
    .rename = lawakfs_rename,
};


int main(int argc, char *argv[]) {
    load_lawak_config("lawak.conf");
    struct stat st;
    if (stat(source_dir, &st) == -1 || !S_ISDIR(st.st_mode)) {
        fprintf(stderr, "Error: Source directory %s does not exist\n", source_dir);
        return 1;
    }

    return fuse_main(argc, argv, &lawakfs_oper, NULL);
}
```

## a. Ekstensi File Tersembunyi

```c
static int hidden_path(char *resolved_path, const char *virtual_path) {
    char virtual_name[MAX_PATH];
    snprintf(virtual_name, sizeof(virtual_name), "%s", virtual_path);
    const char *filename = virtual_name[0] == '/' ? virtual_name + 1 : virtual_name;

    DIR *dp = opendir(source_dir);
    if (!dp) return -errno;

    struct dirent *de;
    while ((de = readdir(dp)) != NULL) {
        if (de->d_name[0] == '.') continue;

        char name_only[MAX_FILENAME];
        strncpy(name_only, de->d_name, MAX_FILENAME - 1);
        name_only[MAX_FILENAME - 1] = '\0';

        char *dot = strrchr(name_only, '.');
        if (dot) *dot = '\0';

        if (strcmp(name_only, filename) == 0) {
            snprintf(resolved_path, MAX_PATH, "%s/%s", source_dir, de->d_name);
            closedir(dp);
            return 0;
        }
    }
```
### Penjelasan:
- Parameter terdapat 2 variabel yaitu, resolved path: Output yang akan diisi dengan path asli file di source_dir dan virtual_path: Input path file virtual dari pengguna, misalnya /laporan
```c
char virtual_name[MAX_PATH];
snprintf(virtual_name, sizeof(virtual_name), "%s", virtual_path);
const char *filename = virtual_name[0] == '/' ? virtual_name + 1 : virtual_name;
```
- Mengambil nama file virtual tanpa /, misalnya dari /laporan jadi laporan
```sh
DIR *dp = opendir(source_dir);
if (!dp) return -errno;
```
- Membuka direktori sumber (tempat file asli disimpan)
```c
while ((de = readdir(dp)) != NULL) {
    if (de->d_name[0] == '.') continue;
```
- Iterasi semua file yang tidak tersembunyi (. atau .. dilewati).
```c
char name_only[MAX_FILENAME];
strncpy(name_only, de->d_name, MAX_FILENAME - 1);
name_only[MAX_FILENAME - 1] = '\0';

char *dot = strrchr(name_only, '.');
if (dot) *dot = '\0';
```
- Mengambil nama file tanpa ekstensi, dengan cara mengambil '.' pada nama file lalu mengganti ekstensi dengan null terminator
```c
if (strcmp(name_only, filename) == 0) {
    snprintf(resolved_path, MAX_PATH, "%s/%s", source_dir, de->d_name);
    closedir(dp);
    return 0;
}
```
- Mengecek nama file yang telah dihilangkan ekstensinya dengan filename asli (ekstensi sudah dihilangkan ketika readdir di akses), jika cocok maka isi resolved_path dengan path lengkap filename asli
```c
closedir(dp);
return -ENOENT;
```
- Jika tidak ditemukan file yang cocok, kembalikan error "No such file or directory".

#### Tambahan:
- Fungsi hidden_path akan dipanggil ketika getattr(), open(), read(), dan readdir() untuk mengecek apakah path sudah cocok ketika ekstensi dihilangkan.
- Alur penerapan:
1. readdir() - Menampilkan Daftar File
Saat user mengetik ls pada mount point, misalnya:
```sh
ls /mnt/LawakFS
```
Fungsi lawakfs_readdir() dipanggil:
```c
char *dot = strrchr(name_copy, '.');
if (dot) *dot = '\0';
filler(buf, name_copy, NULL, 0, 0);
```
- Ini menghapus ekstensi dari nama file yang ditampilkan.
- Hasilnya:
File laporan.txt ditampilkan sebagai laporan.

## 
2. User Mengakses File
Misalnya:
```sh
cat /mnt/LawakFS/laporan
```
- Maka FUSE memanggil lawakfs_open("/laporan"), lawakfs_read("/laporan"), dll.
- Path /laporan yang diterima FUSE sudah tidak punya ekstensi karena:
User tidak pernah tahu nama asli laporan.txt. Yang dia lihat hanya /laporan, dan itu yang dia akses. Maka, siapa yang menyambungkan /laporan ke laporan.txt? Jawabannya adalah fungsi hidden_path()!
```c
if (strcmp(name_only, filename) == 0) {
    snprintf(resolved_path, MAX_PATH, "%s/%s", source_dir, de->d_name);
    ...
}
```
- hidden_path() akan mencocokkan nama virtual tanpa ekstensi (laporan) ke file yang ada di source_dir (misalnya laporan.txt, laporan.pdf, dll).

## b. Akses Berbasis Waktu untuk File Secret
```c
// soal b
static int is_secret_file(const char *filename) {
    const char *base = strrchr(filename, '/');
    base = base ? base + 1 : filename;
    return strncmp(base, secret_basename, strlen(secret_basename)) == 0;
}

// soal b
static int is_accessible_time() {
    time_t now = time(NULL);
    struct tm *tm_now = localtime(&now);
    int hour = tm_now->tm_hour;
    return (hour >= access_start && hour < access_end);
}
```
### Penjelasan:
1. Fungsi is_secret_file(const char *filename) digunakan untuk mengecek apakah file yang diakses memiliki nama dasar "secret" di dalamnya
```c
const char *base = strrchr(filename, '/');
base = base ? base + 1 : filename;
```
- Mencari bagian akhir nama file setelah '/' 
```c
return strncmp(base, secret_basename, strlen(secret_basename)) == 0;
```
- Membandingkan nama dasar dari file yang sedang dibaca dengan secret_basename ("secret")

2. Fungsi is_accessible_time() digunakan untuk mengecek apakah saat ini berada dalam jam akses yang diizinkan untuk mengakses file secret
```c
time_t now = time(NULL);
struct tm *tm_now = localtime(&now);
int hour = tm_now->tm_hour;
```
- Mengambil waktu sekarang dan ekstrak jamnya (0-23)
```c
return (hour >= access_start && hour < access_end);
```
- Periksa apakah jam sekarang termasuk dalam rentang yang diizinkan (accses_start-access_end = 08:00–18:00).

#### Tambahan:

1. Kedua fungsi tersebut akan di implementasikan pada fungsi getattr() yang akan menambahkan kondisi seperti ini:
```c
    if (is_secret_file(path) && !is_accessible_time()) 
        return -ENOENT;
```
- Akan mengembalikan "No such file or directory" ketika file dengan nama dasar secret dibuka pada jam diluar rentang yang diizinkan

2. Pada Fungsi read() di dalamnya terdapat pengecekan kondisi jika file bernama secret dan diakses pada rentang yang diizinkan atau tidak
```c
if (is_secret_file(de->d_name) && !is_accessible_time()) continue;
```
- File dengan nama dasar secret tidak akan ditampilkan dalam daftar file jika diluar jam akses (ketika ls dijalankan)

## c. Filtering Konten Dinamis
```c
char* base64_encode(const unsigned char *input, size_t length, size_t *output_len) {
    BIO *bio, *b64;
    BUF_MEM *bufferPtr;

    b64 = BIO_new(BIO_f_base64());
    bio = BIO_new(BIO_s_mem());
    bio = BIO_push(b64, bio);

    BIO_set_flags(bio, BIO_FLAGS_BASE64_NO_NL);
    BIO_write(bio, input, length);
    BIO_flush(bio);
    BIO_get_mem_ptr(bio, &bufferPtr);
    
    char *encoded = malloc(bufferPtr->length + 1);
    if (!encoded) {
        BIO_free_all(bio);
        return NULL;
    }
    
    memcpy(encoded, bufferPtr->data, bufferPtr->length);
    encoded[bufferPtr->length] = '\0';
    *output_len = bufferPtr->length;
    
    BIO_free_all(bio);
    return encoded;
}

// soal c
static int is_text_file(const char *path) {
    int fd = open(path, O_RDONLY);
    if (fd == -1) return 0;
    
    char buf[1024];
    ssize_t bytes = read(fd, buf, sizeof(buf));
    close(fd);
    
    if (bytes <= 0) return 0;
    
    for (ssize_t i = 0; i < bytes; i++) {
        if (buf[i] == '\0') return 0;
    }
    return 1;
}

static int lawakfs_read(const char *path, char *buf, size_t size, off_t offset,
                        struct fuse_file_info *fi) {
      
  ......

    char *output = NULL;
    size_t output_len = 0;

    if (is_text_file(full_path)) {
        size_t capacity = res * 4; 
        output = malloc(capacity);
        if (!output) {
            free(content);
            close(fd);
            return -ENOMEM;
        }

        size_t input_pos = 0, output_pos = 0;
        while (input_pos < res) {
            int word_matched = 0;

            for (int i = 0; i < lawak_count; ++i) {
                size_t word_len = strlen(lawak_words[i]);

                if (input_pos + word_len <= res &&
                    strncasecmp(&content[input_pos], lawak_words[i], word_len) == 0) {
                    
                    char prev = (input_pos == 0) ? ' ' : content[input_pos - 1];
                    char next = (input_pos + word_len >= res) ? ' ' : content[input_pos + word_len];

                    if (!isalnum(prev) && !isalnum(next)) {
                        memcpy(&output[output_pos], "lawak", 5);
                        output_pos += 5;
                        input_pos += word_len;
                        word_matched = 1;
                        break;
                    }
                }
            }

            if (!word_matched) {
                output[output_pos++] = content[input_pos++];
            }
        }

        if (output_pos > 0 && output[output_pos - 1] != '\n') {
            output[output_pos++] = '\n';
        }

        output[output_pos] = '\0';
        output_len = output_pos;
    } else {
        output = base64_encode((const unsigned char *)content, res, &output_len);
        if (!output) {
            free(content);
            close(fd);
            return -ENOMEM;
        }
    }

    size_t bytes_to_copy = 0;
    if (offset < output_len) {
        bytes_to_copy = (offset + size > output_len) ? output_len - offset : size;
        memcpy(buf, output + offset, bytes_to_copy);
    }

    free(content);
    free(output);
    close(fd);

    log_action("READ", path);
    return bytes_to_copy;
}
```
### Penjelasan: 
1. Fungsi is_text_file() berfungsi untuk mendeteksi tipe file teks atau biner
```c
static int is_text_file(const char *path) {
    int fd = open(path, O_RDONLY);
    if (fd == -1) return 0;
    
    char buf[1024];
    ssize_t bytes = read(fd, buf, sizeof(buf));
    close(fd);
    
    if (bytes <= 0) return 0;

    for (ssize_t i = 0; i < bytes; i++) {
        if (buf[i] == '\0') return 0;
    }
    return 1;
}
```
- open() file dalam mode baca (O_RDONLY), jika gagal dibuka langsung dianggap bukan file teks (return 0)
- read() sebanyak maksimal 1024 byte dari awal file, jika tidak bisa dibaca atau kosong return 0.
- loop setiap byte dari buffer yang dibaca, jika ada null byte '\0' maka akan diasumsikan ini bukan file teks
- return 0 jika file tersebut adalah biner dan return 1 jika file adalah text file

2.  Fungsi base64_encode() berfungsi untuk meng-encode konten yang berjenis file biner
```c
char* base64_encode(const unsigned char *input, size_t length, size_t *output_len) {
    BIO *bio, *b64;
    BUF_MEM *bufferPtr;

    b64 = BIO_new(BIO_f_base64());
    bio = BIO_new(BIO_s_mem());
    bio = BIO_push(b64, bio);

    BIO_set_flags(bio, BIO_FLAGS_BASE64_NO_NL);
    BIO_write(bio, input, length);
    BIO_flush(bio);
    BIO_get_mem_ptr(bio, &bufferPtr);

    char *encoded = malloc(bufferPtr->length + 1);
    if (!encoded) {
        BIO_free_all(bio);
        return NULL;
    }

    memcpy(encoded, bufferPtr->data, bufferPtr->length);
    encoded[bufferPtr->length] = '\0';
    *output_len = bufferPtr->length;

    BIO_free_all(bio);
    return encoded;
}
```
Membuat encoder base64 dari OpenSSL
- BIO_f_base64() = filter untuk encode base64
- BIO_s_mem() = output ke memory
- BIO_push() = gabungkan filter dan buffer
Mematikan newwline default
- BIO_set_flags(bio, BIO_FLAGS_BASE64_NO_NL) berfungsi untuk mematikan newline default (newline tiap 64 karakter)
Proses Encode Data
- BIO_write(bio, input, length) berfungsi untuk menulis isi file
- BIO_flush(bio) berfungsi untuk memastikan bahwa semua data telah masuk
- BIO_get_mem_ptr(bio, &bufferPtr) mengambil hasil encoding dalam buffer
  alin hasil ke malloc buffer biasa
```c
char *encoded = malloc(bufferPtr->length + 1);
memcpy(encoded, bufferPtr->data, bufferPtr->length);
encoded[bufferPtr->length] = '\0';
```
- Menyalin buffer yang telah di encode dengan OpenSSL ke buffer sendiri, agar dapat digunakan diluat OpenSSL
- return buffer yang telah disalin dan free memori bio

3. Pada fungsi lawakfs_read() sebagian besar perubahan berada dibawah kondisi if(is_text_file)
- Dilakukan pengecekan jika file terdeteksi sebagai file text atau tidak
```c
size_t capacity = res * 4;
output = malloc(capacity);
```
- Mengalokasikan buffer output yang cukup besar sebanyak 4x dari input untuk jaga jaga adanya kata pendek yang diganti dengan huruf "lawak"
```c
size_t input_pos = 0, output_pos = 0;
while (input_pos < res) {
    int word_matched = 0;
```
- Menginisialisasi size dari indeks yang dibaca (input_pos) dan indeks yang akan ditulis(output_pos)
- Loop hingga semua karakter dari konten dibaca.
```c
while (input_pos < res){
...
```
- Mengiterasi setiap huruf pada input
```c
for (int i = 0; i < lawak_count; ++i) {
    size_t word_len = strlen(lawak_words[i]);
```
- Loop semua kata dari konfigurasi yang harus difilter (lawak_words[])
```c
if (input_pos + word_len <= res &&
    strncasecmp(&content[input_pos], lawak_words[i], word_len) == 0) {
```
- Cek apakah kata yang sedang dibaca cocok dengan salah satu kata dari lawak_words[]
- strncasecmp berfungsi membandingkan tanpa memperhatikan huruf besar/kecil.
```c
char prev = (input_pos == 0) ? ' ' : content[input_pos - 1];
char next = (input_pos + word_len >= res) ? ' ' : content[input_pos + word_len];
```
- Cek apakah kata yang dicocokkan merupakan kata utuh, bukan merupakab bagian dari kata lain
- Caranya dengan mengecek apakah ada huruf/angka di 1 indeks sebelum dan sesudah kata tersebut
```c
if (!isalnum(prev) && !isalnum(next)) {
memcpy(&output[output_pos], "lawak", 5);
output_pos += 5;
input_pos += word_len;
word_matched = 1;
break;
}
```
- Jika previous dan next tidak terdapat huruf/angka lain maka kata yang sedang dicek akan diganti dengan kata "lawak"
- Salin string "lawak" ke output
- lewati input_pos sejauh panjang kata yang dicocokkan
```c
if (!word_matched) {
    output[output_pos++] = content[input_pos++];
}
```
- Jika kata tidak cocok, maka salin karakter seperti biasa pada output_pos
```c
if (output_pos > 0 && output[output_pos - 1] != '\n') {
    output[output_pos++] = '\n';
}
```
- Menambahkan newline jika tidak ada di akhir
```c
output[output_pos] = '\0';
output_len = output_pos;
```
- Null terminate output dan simpan panjang output akhir
```c
} else {
    output = base64_encode((const unsigned char *)content, res, &output_len);
    if (!output) {
        free(content);
        close(fd);
        return -ENOMEM;
    }
}
```
- JIka file yang dibaca adalah file biner maka encode isinya menjadi base64
- Simpan hasilnya pada output
```c
size_t bytes_to_copy = 0;
if (offset < output_len) {
    bytes_to_copy = (offset + size > output_len) ? output_len - offset : size;
    memcpy(buf, output + offset, bytes_to_copy);
}
```
- Menangani pembacaan sebagian (offset), seperti permintaan baca dari FUSE, lalu salin isi output ke buf, maksimal size byte
```c
free(content);
free(output);
close(fd);

log_action("READ", path);
return bytes_to_copy;
```
- Bebaskan memori dan file descriptor
- Log aktivitas READ ke log file lawakfs.log
- Kembalikan jumlah byte yang berhasil dibaca

## d. Logging Akses
```c
#define LOG_PATH "/home/ferdian-ardra/praktikum_linux/modul4/sisop-modul-4-mitrapartogi13/task-2/var/log/lawakfs.log"

void log_action(const char *action, const char *path) {
    FILE *log_file = fopen(LOG_PATH, "a");
    if (!log_file) return;

    time_t t = time(NULL);
    struct tm tm_info;
    localtime_r(&t, &tm_info);

    char timestamp[20];
    strftime(timestamp, sizeof(timestamp), "%F %T", &tm_info);

    uid_t uid = getuid();

    fprintf(log_file, "[%s] [%d] [%s] [%s]\n", timestamp, uid, action, path);
    fclose(log_file);
}
```
### Penjelasan:
- Membuka file log dengan fopen() sesuai dengan path yang diberikan
- jika file error gagal dibuka, return
- Mengambil waktu saat ini (timestamp) menggunakan localtime_r()
- Mengubah struktur tm_info menjadi string dengan format YYYY-MM-DD HH:MM:SS
- Mengambil user ID dari pengguna yang menjalankan operasi fuse
- Menulis 1 barus log ke file lawakfs.log
- Terapkan pada fungsi read() dan access()

## e. Konfigurasi
```c
char lawak_words[MAX_LAWAK_WORDS][MAX_WORD_LEN];
int lawak_count = 0;
char secret_basename[MAX_FILENAME] = "secret";
int access_start = 8;    // 08:00
int access_end = 18;     // 18:00

void load_lawak_config(const char *config_path) {
    printf("[CONFIG] Loading configuration from: %s\n", config_path);
    
    FILE *fp = fopen(config_path, "r");
    if (!fp) {
        printf("[WARNING] Could not open config file %s: %s\n", config_path, strerror(errno));
        printf("[CONFIG] Using default values\n");
        return;
    }

    char line[MAX_WORD_LEN * 2]; 
    while (fgets(line, sizeof(line), fp)) {
        line[strcspn(line, "\n")] = '\0';
        if (line[0] == '\0' || line[0] == '#') continue;

        char *key = strtok(line, "=");
        char *value = strtok(NULL, "=");
        
        if (!key || !value) continue;

        if (strcmp(key, "SECRET_FILE_BASENAME") == 0) {
            strncpy(secret_basename, value, MAX_FILENAME - 1);
            secret_basename[MAX_FILENAME - 1] = '\0';
            printf("[CONFIG] Set secret_basename: %s\n", secret_basename);
        } 
        else if (strcmp(key, "access_start") == 0) {
            access_start = atoi(value);
            printf("[CONFIG] Set access_start: %02d:00\n", access_start);
        }
        else if (strcmp(key, "access_end") == 0) {
            access_end = atoi(value);
            printf("[CONFIG] Set access_end: %02d:00\n", access_end);
        }
        else if (strcmp(key, "lawak_words") == 0) {
            char *word = strtok(value, ",");
            while (word && lawak_count < MAX_LAWAK_WORDS) {
                while (isspace(*word)) word++;
                char *end = word + strlen(word) - 1;
                while (end > word && isspace(*end)) end--;
                *(end + 1) = '\0';

                if (*word) {
                    strncpy(lawak_words[lawak_count], word, MAX_WORD_LEN - 1);
                    lawak_words[lawak_count][MAX_WORD_LEN - 1] = '\0';
                    printf("[CONFIG] Added lawak word: %s\n", lawak_words[lawak_count]);
                    lawak_count++;
                }
                word = strtok(NULL, ",");
            }
        }
    }
    fclose(fp);
}
```
```log
// lawak.conf
lawak_words=ducati,ferrari,mu,chelsea,prx,onic,sisop
SECRET_FILE_BASENAME=secret
access_start=08
access_end=18
```
### Penjelasan:
- Mencetak pesan debug "loading configuration from: ..."
- Membuka file config sesuai dengan path yang diberikan untuk dibaca
```c
while (fgets(line, sizeof(line), fp)) {
  line[strcspn(line, "\n")] = '\0';
  if (line[0] == '\0' || line[0] == '#') continue;
  ...
```
- Membaca file baris demi baris
- Menghapus neline di akhir baris
- Melewati baris kosong atau komentar
```c
  char *key = strtok(line, "=");
  char *value = strtok(NULL, "=");
  if (!key || !value) continue;

  if (strcmp(key, "SECRET_FILE_BASENAME") == 0) {
    strncpy(secret_basename, value, MAX_FILENAME - 1);
    secret_basename[MAX_FILENAME - 1] = '\0';
  else if (strcmp(key, "access_start") == 0) {
      access_start = atoi(value);
      printf("[CONFIG] Set access_start: %02d:00\n", access_start);
  }
  else if (strcmp(key, "access_end") == 0) {
      access_end = atoi(value);
      printf("[CONFIG] Set access_end: %02d:00\n", access_end);
  }
```
- Memisahkan string berdasarkan '=' yang akan menghasilkan pasangan key = value
- Melewati baris yang fdormatnya tidak valid
- Menyimpan nama file secret sesuai dengan secret_basename pada config
- Mengubah string pada key yang diakses pada config menjadi integer, lalu disimpan nilainya pada access_start dan access_end
```c
  else if (strcmp(key, "lawak_words") == 0) {
      char *word = strtok(value, ",");
      while (word && lawak_count < MAX_LAWAK_WORDS) {
          while (isspace(*word)) word++;
          char *end = word + strlen(word) - 1;
          while (end > word && isspace(*end)) end--;
          *(end + 1) = '\0';

          if (*word) {
              strncpy(lawak_words[lawak_count], word, MAX_WORD_LEN - 1);
              lawak_words[lawak_count][MAX_WORD_LEN - 1] = '\0';
              printf("[CONFIG] Added lawak word: %s\n", lawak_words[lawak_count]);
              lawak_count++;
          }
          word = strtok(NULL, ",");
      }
  }
```
- Mengecek jika keynya adalah "lawak_words", maka nilai akan dipisah berdasarkan koma
- isspace() digunakan untuk membersihkan spasi di awal dan akhir kata.
- Kata valid dimasukkan ke array global lawak_words[][].
- lawak_count diincrement untuk melacak berapa jumlah kata yang disimpan.
#### Tambahan:
- Fungsi load_lawak_config akan dijalankan pada int main() dengan seperti ini "load_lawak_config("lawak.conf");"
- Semua variabel yang diakses dari lawan.conf nantinya akan dipakai pada fungsi fungsi yang membutuhkan seperti is_secret_file, is_accessible_time, dan fungsi yang membutuhkan lawak_words[] yang nanti akan diganti dengan kata lawak ketika dibaca.




<h1>Task 3 - Drama Troll </h1>

## a. Pembuatan User

```sh

sudo useradd -m DainTontas
sudo passwd DainTontas # pw : DainTontas

sudo useradd -m Ryeku
sudo passwd Ryeku # pw : Ryeku

sudo useradd -m SunnyBolt
sudo passwd SunnyBolt # pw : SunnyBolt

```
### Penjelasan: 

sudo useradd -m x : menambah user beserta direktorinya masing-masing
maka dari kode di atas akan membuat user baru yaitu DainTontas, Ryeku, dan SunnyBolt beserta home directorynya.

https://github.com/user-attachments/assets/5f384ada-0914-47ba-8e1a-54ce2abf3d38

## b. Jebakan Troll
Diminta pada soal membuat kode FUSE yaitu troll.c yang berisi 2 file sederhana yaitu very_spicy_info.txt dan upload.txt
kode :

```c
static int troll_readdir(const char *path, void *buf, fuse_fill_dir_t filler,
                         off_t offset, struct fuse_file_info *fi,
                         enum fuse_readdir_flags flags) {
    (void) offset;
    (void) fi;
    (void) flags;

    if (strcmp(path, "/") != 0)
        return -ENOENT;

    filler(buf, ".", NULL, 0, 0);
    filler(buf, "..", NULL, 0, 0);
    filler(buf, "very_spicy_info.txt", NULL, 0, 0);
    filler(buf, "upload.txt", NULL, 0, 0);
    return 0;
}
```
### Penjelasan :

- Fungsi troll_readdir dipanggil saat isi direktori dibaca, misalnya saat pengguna menjalankan ls /mnt/troll hanya ketika apabila sudah mount.
- Fungsi ini menggunakan filler(...) untuk mengisi daftar isi direktori.

## c. Jebakan Troll (Berlanjut)
Soal ini meminta apabila ketika user selain DainTontas membuka very_spicy_info.txt menggunakan perintah cat akan memunculkan DainTontas' `personal secret!!.txt` dan apabila DainTontas yang membuka akan muncul `Very spicy internal developer information: leaked roadmap.docx` 

Sebelum itu, supaya bisa dijalankan user lain, kita harus allow other user untuk mengakses file di /mnt/troll dengan cara
```sh
sudo nano /etc/fuse.conf
# lalu un-comment yang ada tulisan "user_allow_other"
```
lalu chmod, sudo chmod 777 /mnt/troll


lalu compile file FUSE kita (troll.c) gcc -Wall troll.c `pkg-config fuse3 --cflags --libs` -o troll
lalu mount menggunakan ./troll -o allow_other /mnt/troll
untuk unmount gunakan sudo fusermount3 -u /mnt/troll
untuk ganti user bisa su - x (dimana x adalah username tujuan yang mau diganti)

cek apakah user lain bisa mengakses file di /mnt/troll



lalu, kita masuk ke inti penyelesaian soal
```c
static int troll_read(const char *path, char *buf, size_t size, off_t offset,
                      struct fuse_file_info *fi) {
    size_t len;
    const char *content;

    if (strcmp(path, "/very_spicy_info.txt") == 0) {
        uid_t uid = fuse_get_context()->uid;
        struct passwd *pw = getpwuid(uid);

        if (pw && strcmp(pw->pw_name, "DainTontas") == 0) {
            content = "Very spicy internal developer information: leaked roadmap.docx";
        } else {
            content = "DainTontas' personal secret!!.txt";
        }
    } else if (strcmp(path, "/upload.txt") == 0) {
        content = "";
    } else {
        return -ENOENT;
    }

    len = strlen(content);
    if (offset < len) {
        if (offset + size > len)
            size = len - offset;
        memcpy(buf, content + offset, size);
    } else
        size = 0;

    return size;
}

```
### Penjelasan :
``` c
if (pw && strcmp(pw->pw_name, "DainTontas") == 0) {
            content = "Very spicy internal developer information: leaked roadmap.docx";
        } else {
            content = "DainTontas' personal secret!!.txt";
        }
```
 - Disini dibuktikan apabila user yg mengakses adalah DainTontas, maka akan muncul `Very spicy internal developer information: leaked roadmap.docx`
   dan apabila bukan akan muncul `DainTontas' personal secret!!.txt`.

Jangan lupa setelah update/merubah kode FUSE/ kode troll.c supaya bisa berjalan sesuai dengan keinginan/kebutuhan kita, kita perlu compile ulang
`gcc -Wall troll.c `pkg-config fuse3 --cflags --libs` -o troll`, lalu di unmount `sudo fusermount3 -u /mnt/troll`, terakhir di mount ulang `./troll -o allow_other /mnt/troll`


## d. Trap
Soal ini meminta apabila DainTontas melakukan echo ke upload.txt (`echo upload > /mnt/troll/upload.txt`) akan menyalakan sebuah trigger yang membuat semua user ketika membuka file yang berekstensi `.txt` akan menampilakn ASCII Art dari kalimat `Fell for it again reward`.

dan tambahan dari soal, meskipun mesin dinyalakan ulang atau unmount, trigger ini akan tetap ada.

#### 1. Pemicu Jebakan 
```c
static int troll_write(const char *path, const char *buf, size_t size, off_t offset,
                       struct fuse_file_info *fi) {
    if (strcmp(path, "/upload.txt") == 0) {
        FILE *f = fopen(trigger_path, "w");
        if (f) {
            fprintf(f, "trap triggered\n");
            fclose(f);
        }
        return size;
    }
    return -EACCES;
}

```
### Penjelasan :
1. Saat file upload.txt ditulis (path == "/upload.txt")
2. Maka akan :
   - Membuka file trigger_path (file /tmp/.troll_triggered) dalam mode write ("w"),
   - Kalau berhasil dibuka (if (f)),
     - Menuliskan string trap triggered\n ke file itu.
     - Menutup file.
   - Setelah itu, dia akan mengembalikan nilai size (artinya operasi write dianggap berhasil).
  
Penjelasan lebih praktikal/kenapa bisa `echo "upload" > /mnt/troll/upload.txt` memicu trigger :
- Apa yang dilakukan echo di balik layar?
   - echo menulis string "upload" ke file /mnt/troll/upload.txt.
   - Kernel menjalankan syscall write() pada file FUSE kita.
   - Karena kita sudah register fungsi .write di FUSE menjadi troll_write(), maka fungsi ini terpanggil.

- Di dalam troll_write():
   - Dicek path-nya => benar /upload.txt.
   - Maka file /tmp/.troll_triggered dibuat/ditulis ulang.
   - in other words: trIGGER aktif
 
  






