# 一些libc工具

- glibc_all_in_one
- patchelf
- libcsearch
  
  ***

## glibc_all_in_one与patchelf配置

glibc-all-in-one:

```sh
sudo git clone https://github.com/matrix1001/glibc-all-in-one.git
cd glibc-all-in-one/
cat list #有的libc列表，较老版本的libc在old_list
sudo ./download xxx #下载需要的对应版本的libc,较老版本的libc用download_old
```

patchelf(用于将对应程序绑定到对应版本libc)

```sh
sudo apt install patchelf
```

## glibc_all_in_one与patchelf使用

题目给libc文件，先看一下是什么版本

```sh
strings ./libc-xx.x | grep ubuntu
```

e.g.

```sh
┌──(kali㉿helloeveryone)-[~/ctf/pwn/q28_int]
└─$ strings ./libc.so.6 | grep ubuntu
GNU C Library (Ubuntu GLIBC 2.23-0ubuntu11.3) stable release version 2.23, by Roland McGrath et al.
<https://bugs.launchpad.net/ubuntu/+source/glibc/+bugs>.
```

然后查找下载对应libc(用glibc-all-in-one)

```sh
┌──(kali㉿helloeveryone)-[~/glibc-all-in-one]
└─$ cat list
2.23-0ubuntu11.3_amd64
2.23-0ubuntu11.3_i386
2.23-0ubuntu3_amd64
2.23-0ubuntu3_i386
2.27-3ubuntu1.5_amd64
2.27-3ubuntu1.5_i386
2.27-3ubuntu1.6_amd64
2.27-3ubuntu1.6_i386
2.27-3ubuntu1_amd64
2.27-3ubuntu1_i386
2.31-0ubuntu9.14_amd64
2.31-0ubuntu9.14_i386
2.31-0ubuntu9_amd64
2.31-0ubuntu9_i386
2.35-0ubuntu3.6_amd64
2.35-0ubuntu3.6_i386
2.35-0ubuntu3_amd64
2.35-0ubuntu3_i386
2.37-0ubuntu2.2_amd64
2.37-0ubuntu2.2_i386
2.37-0ubuntu2_amd64
2.37-0ubuntu2_i386
2.38-1ubuntu6_amd64
2.38-1ubuntu6_i386
2.38-3ubuntu1_amd64
2.38-3ubuntu1_i386
┌──(kali㉿helloeveryone)-[~/glibc-all-in-one]
└─$ sudo ./download 2.23-0ubuntu11.3_i386
...
```

下载好后如下

```sh
┌──(kali㉿helloeveryone)-[~/glibc-all-in-one/libs/2.23-0ubuntu11.3_i386]
└─$ ls
ld-2.23.so               libcrypt-2.23.so  libnsl.so.1            libnss_nis-2.23.so      librt-2.23.so
ld-linux.so.2            libcrypt.so.1     libnss_compat-2.23.so  libnss_nisplus-2.23.so  librt.so.1
libanl-2.23.so           libc.so.6         libnss_compat.so.2     libnss_nisplus.so.2     libSegFault.so
libanl.so.1              libdl-2.23.so     libnss_dns-2.23.so     libnss_nis.so.2         libthread_db-1.0.so
libBrokenLocale-2.23.so  libdl.so.2        libnss_dns.so.2        libpcprofile.so         libthread_db.so.1
libBrokenLocale.so.1     libm-2.23.so      libnss_files-2.23.so   libpthread-2.23.so      libutil-2.23.so
libc-2.23.so             libmemusage.so    libnss_files.so.2      libpthread.so.0         libutil.so.1
libcidn-2.23.so          libm.so.6         libnss_hesiod-2.23.so  libresolv-2.23.so
libcidn.so.1             libnsl-2.23.so    libnss_hesiod.so.2     libresolv.so.2
```

然后用patchelf配置对应libc

```sh
cd xxx #题目目录
patchelf --set-interpreter ~/glibc-all-in-one/libs/2.23-0ubuntu11.3_i386/ld-2.23.so ./xxxx
#配置链接器到题目xxxx
patchelf --set-rpath ~/glibc-all-in-one/libs/2.23-0ubuntu11.3_i386/ ./xxxx
#配置libc加载目录到题目xxxx
```

现在就可以用ldd查看以来是否改变了

```sh
ldd ./xxxx
```

## libcsearch

未完




