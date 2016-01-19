# linux-from-scratch-7.7-systemd-with-ubuntu14.04
7.7-systemd版本的linux from scratch的一次实践，宿主系统是ubuntu14.04，硬件是amd的物理主机上的虚拟机：virtual-box

##宿主系统预备
【ubuntu 14.04环境设置（输入）】
```shell
sudo rm /bin/sh
sudo ln -s /bin/bash /bin/sh
sudo apt-get update
sudo apt-get install bison gawk build-essential texinfo
```
【检查宿主系统需求（输入）】
```shell
cat > version-check.sh << "EOF"
#!/bin/bash
# Simple script to list version numbers of critical development tools
export LC_ALL=C
bash --version | head -n1 | cut -d" " -f2-4
echo "/bin/sh -> `readlink -f /bin/sh`"
echo -n "Binutils: "; ld --version | head -n1 | cut -d" " -f3-
bison --version | head -n1
if [ -h /usr/bin/yacc ]; then
echo "/usr/bin/yacc -> `readlink -f /usr/bin/yacc`";
elif [ -x /usr/bin/yacc ]; then
echo yacc is `/usr/bin/yacc --version | head -n1`
else
echo "yacc not found"
fi
bzip2 --version 2>&1 < /dev/null | head -n1 | cut -d" " -f1,6-
echo -n "Coreutils: "; chown --version | head -n1 | cut -d")" -f2
diff --version | head -n1
find --version | head -n1
gawk --version | head -n1
if [ -h /usr/bin/awk ]; then
echo "/usr/bin/awk -> `readlink -f /usr/bin/awk`";
elif [ -x /usr/bin/awk ]; then
echo yacc is `/usr/bin/awk --version | head -n1`
else
echo "awk not found"
fi
gcc --version | head -n1
g++ --version | head -n1
ldd --version | head -n1 | cut -d" " -f2- # glibc version
grep --version | head -n1
gzip --version | head -n1
cat /proc/version
m4 --version | head -n1
make --version | head -n1
patch --version | head -n1
echo Perl `perl -V:version`
sed --version | head -n1
tar --version | head -n1
makeinfo --version | head -n1
xz --version | head -n1
echo 'main(){}' > dummy.c && g++ -o dummy dummy.c
if [ -x dummy ]
then echo "g++ compilation OK";
else echo "g++ compilation failed"; fi
rm -f dummy.c dummy
EOF
```
【检查宿主系统需求（输出参考）】
```shell
ubuntu@ubuntu:~$ bash version-check.sh
bash, version 4.3.11(1)-release
/bin/sh -> /bin/bash
Binutils: (GNU Binutils for Ubuntu) 2.24
bison (GNU Bison) 3.0.2
/usr/bin/yacc -> /usr/bin/bison.yacc
bzip2,  Version 1.0.6, 6-Sept-2010.
Coreutils:  8.21
diff (GNU diffutils) 3.3
find (GNU findutils) 4.4.2
GNU Awk 4.0.1
/usr/bin/awk -> /usr/bin/gawk
gcc (Ubuntu 4.8.4-2ubuntu1~14.04) 4.8.4
g++ (Ubuntu 4.8.4-2ubuntu1~14.04) 4.8.4
(Ubuntu EGLIBC 2.19-0ubuntu6.6) 2.19
grep (GNU grep) 2.16
gzip 1.6
Linux version 3.19.0-25-generic (buildd@lgw01-20) (gcc version 4.8.2 (Ubuntu 4.8.2-19ubuntu1) ) #26~14.04.1-Ubuntu SMP Fri Jul 24 21:16:20 UTC 2015
m4 (GNU M4) 1.4.17
GNU Make 3.81
GNU patch 2.7.1
Perl version='5.18.2';
sed (GNU sed) 4.2.2
tar (GNU tar) 1.27.1
makeinfo (GNU texinfo) 5.2
xz (XZ Utils) 5.1.0alpha
g++ compilation OK
ubuntu@ubuntu:~$
```
【库文件的一致性检查（输入）】
```shell
cat > library-check.sh << "EOF"
#!/bin/bash
for lib in lib{gmp,mpfr,mpc}.la; do
echo $lib: $(if find /usr/lib* -name $lib|
grep -q $lib;then :;else echo not;fi) found
done
unset lib
EOF
```
【库文件的一致性检查（输出参考）】
```shell
ubuntu@ubuntu:~$ bash library-check.sh
libgmp.la: not found
libmpfr.la: not found
libmpc.la: not found
ubuntu@ubuntu:~$
```
这些文件应该要么都在或者是都缺失，而不应该只有一两个。

##构建前准备

【分区】
```shell
sudo fdisk -l
sudo mkfs -v -t ext4 /dev/sda
```
【在分区上创建文件系统】
```shell
export LFS=/mnt/lfs

sudo mkdir -pv $LFS   #建立挂载点
sudo mount -v -t ext4 /dev/sda $LFS   #将/dev/sda挂载到$LFS
```
【设置 $LFS 变量】
```shell
sudo passwd root
su -
export LFS=/mnt/lfs
cat > /root/.bash_profile << "EOF"
#!/bin/bash
export LFS=/mnt/lfs
EOF
```
【创建$LFS/tools文件夹】
```shell
sudo ln -sv $LFS/tools /
sudo mkdir -v $LFS/sources
sudo chmod -v a+wt $LFS/sources
```
在宿主系统中创建/tools的符号链接,将其指向LFS分区中新建的文件夹，输出的提示是："/tools" -> "/mnt/lfs/tools"

【添加LFS用户】
```shell
sudo groupadd lfs
sudo useradd -s /bin/bash -g lfs -m -k /dev/null lfs
sudo passwd lfs
sudo chown -v lfs $LFS/tools
sudo chown -v lfs $LFS/sources
```
【设置环境】
```shell
su - lfs
cat > ~/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
source ~/.bash_profile
EOF
```
exec env -i.../bin/bash 命令用一个除了HOME 、 TERM 和 PS1 变量,完全空环境的 shell 代替运行中的 shell。
```shell
cat > ~/.bashrc << "EOF"
set +h
umask 022
LFS=/mnt/lfs
LC_ALL=POSIX
LFS_TGT=$(uname -m)-lfs-linux-gnu
PATH=/tools/bin:/bin:/usr/bin
export LFS LC_ALL LFS_TGT PATH
EOF

source ~/.bash_profile
```
【软件包与补丁】
```shell
cd $LFS/sources
wget 'https://linux.cn/lfs/LFS-BOOK-7.7-systemd/wget-list'
wget --input-file=wget-list --continue --directory-prefix=$LFS/sources

pushd $LFS/sources
wget 'https://linux.cn/lfs/LFS-BOOK-7.7-systemd/md5sums'
md5sum -c md5sums
popd
```

##构建临时系统
【Binutils-2.25 - 第一遍（安装交叉编译的 Binutils）】
```shell
cd $LFS/sources
tar -jxvf binutils-2.25.tar.bz2
cd binutils-2.25
mkdir -v ../binutils-build
cd ../binutils-build

../binutils-2.25/configure     \
    --prefix=/tools            \
    --with-sysroot=$LFS        \
    --with-lib-path=/tools/lib \
    --target=$LFS_TGT          \
    --disable-nls              \
    --disable-werror

make

case $(uname -m) in
  x86_64) mkdir -v /tools/lib && ln -sv lib /tools/lib64 ;;
esac

make install

cd $LFS/sources
rm -rf binutils-build/ binutils-2.25/
```
【GCC-4.9.2 - 第一遍（安装交叉编译的 GCC）】
```shell
cd $LFS/sources
tar -jxvf gcc-4.9.2.tar.bz2
cd gcc-4.9.2

tar -xf ../mpfr-3.1.2.tar.xz
mv -v mpfr-3.1.2 mpfr
tar -xf ../gmp-6.0.0a.tar.xz
mv -v gmp-6.0.0 gmp
tar -xf ../mpc-1.0.2.tar.gz
mv -v mpc-1.0.2 mpc

for file in \
 $(find gcc/config -name linux64.h -o -name linux.h -o -name sysv4.h)
do
  cp -uv $file{,.orig}
  sed -e 's@/lib\(64\)\?\(32\)\?/ld@/tools&@g' \
      -e 's@/usr@/tools@g' $file.orig > $file
  echo '
#undef STANDARD_STARTFILE_PREFIX_1
#undef STANDARD_STARTFILE_PREFIX_2
#define STANDARD_STARTFILE_PREFIX_1 "/tools/lib/"
#define STANDARD_STARTFILE_PREFIX_2 ""' >> $file
  touch $file.orig
done

sed -i '/k prot/agcc_cv_libc_provides_ssp=yes' gcc/configure

mkdir -v ../gcc-build
cd ../gcc-build

../gcc-4.9.2/configure                             \
    --target=$LFS_TGT                              \
    --prefix=/tools                                \
    --with-sysroot=$LFS                            \
    --with-newlib                                  \
    --without-headers                              \
    --with-local-prefix=/tools                     \
    --with-native-system-header-dir=/tools/include \
    --disable-nls                                  \
    --disable-shared                               \
    --disable-multilib                             \
    --disable-decimal-float                        \
    --disable-threads                              \
    --disable-libatomic                            \
    --disable-libgomp                              \
    --disable-libitm                               \
    --disable-libquadmath                          \
    --disable-libsanitizer                         \
    --disable-libssp                               \
    --disable-libvtv                               \
    --disable-libcilkrts                           \
    --disable-libstdc++-v3                         \
    --enable-languages=c,c++

make
make install

cd $LFS/sources
rm -rf gcc-build/ gcc-4.9.2/
```
【Linux-3.19 API 头文件】
```shell
cd $LFS/sources
tar -xvf linux-3.19.tar.xz
cd linux-3.19
make mrproper
make INSTALL_HDR_PATH=dest headers_install
cp -rv dest/include/* /tools/include

cd $LFS/sources
rm -rf linux-3.19/
```
【Glibc-2.21】
```shell
cd $LFS/sources
tar -xvf glibc-2.21.tar.xz
cd glibc-2.21

if [ ! -r /usr/include/rpc/types.h ]; then
  su -c 'mkdir -pv /usr/include/rpc'
  su -c 'cp -v sunrpc/rpc/*.h /usr/include/rpc'
fi

sed -e '/ia32/s/^/1:/' \
    -e '/SSE2/s/^1://' \
    -i  sysdeps/i386/i686/multiarch/mempcpy_chk.S

mkdir -v ../glibc-build
cd ../glibc-build

../glibc-2.21/configure                             \
      --prefix=/tools                               \
      --host=$LFS_TGT                               \
      --build=$(../glibc-2.21/scripts/config.guess) \
      --disable-profile                             \
      --enable-kernel=2.6.32                        \
      --with-headers=/tools/include                 \
      libc_cv_forced_unwind=yes                     \
      libc_cv_ctors_header=yes                      \
      libc_cv_c_cleanup=yes

make

make install

cd $LFS/sources
rm -rf gcc-build/ glibc-2.21/
```
【确认新工具链的基本功能(编译和链接)都是像预期的那样正常工作】
```shell
echo 'main(){}' > dummy.c
$LFS_TGT-gcc dummy.c
readelf -l a.out | grep ': /tools'
```
如果一切工作正常的话，这里应该没有错误，最后一个命令的输出形式会是：
[Requesting program interpreter: /tools/lib/ld-linux.so.2]
```shell
rm -v dummy.c a.out
```
【Libstdc++-4.9.2】
```shell
cd $LFS/sources
tar -jxvf gcc-4.9.2.tar.bz2
cd gcc-4.9.2

mkdir -pv ../gcc-build
cd ../gcc-build

../gcc-4.9.2/libstdc++-v3/configure \
    --host=$LFS_TGT                 \
    --prefix=/tools                 \
    --disable-multilib              \
    --disable-shared                \
    --disable-nls                   \
    --disable-libstdcxx-threads     \
    --disable-libstdcxx-pch         \
    --with-gxx-include-dir=/tools/$LFS_TGT/include/c++/4.9.2

make

make install 

cd $LFS/sources
rm -rf gcc-build/ gcc-4.9.2/
```
【Binutils-2.25 - 第2遍】
```shell
cd $LFS/sources
rm -rf binutils-build binutils-2.25

cd $LFS/sources
tar xf binutils-2.25.tar.bz2
cd binutils-2.25
mkdir -v ../binutils-build
cd ../binutils-build

CC=$LFS_TGT-gcc                \
AR=$LFS_TGT-ar                 \
RANLIB=$LFS_TGT-ranlib         \
../binutils-2.25/configure     \
    --prefix=/tools            \
    --disable-nls              \
    --disable-werror           \
    --with-lib-path=/tools/lib \
    --with-sysroot

make

make install

make -C ld clean
make -C ld LIB_PATH=/usr/lib:/lib
cp -v ld/ld-new /tools/bin

cd $LFS/sources
rm -rf binutils-build binutils-2.25
```
【GCC-4.9.2 - 第2遍】
```shell
cd $LFS/sources
rm -rf gcc-4.9.2 gcc-build
tar -xvf gcc-4.9.2.tar.bz2
cd gcc-4.9.2

cat gcc/limitx.h gcc/glimits.h gcc/limity.h > \
  `dirname $($LFS_TGT-gcc -print-libgcc-file-name)`/include-fixed/limits.h

for file in \
 $(find gcc/config -name linux64.h -o -name linux.h -o -name sysv4.h)
do
  cp -uv $file{,.orig}
  sed -e 's@/lib\(64\)\?\(32\)\?/ld@/tools&@g' \
      -e 's@/usr@/tools@g' $file.orig > $file
  echo '
#undef STANDARD_STARTFILE_PREFIX_1
#undef STANDARD_STARTFILE_PREFIX_2
#define STANDARD_STARTFILE_PREFIX_1 "/tools/lib/"
#define STANDARD_STARTFILE_PREFIX_2 ""' >> $file
  touch $file.orig
done

tar -xf ../mpfr-3.1.2.tar.xz
mv -v mpfr-3.1.2 mpfr
tar -xf ../gmp-6.0.0a.tar.xz
mv -v gmp-6.0.0 gmp
tar -xf ../mpc-1.0.2.tar.gz
mv -v mpc-1.0.2 mpc

mkdir -v ../gcc-build
cd ../gcc-build

CC=$LFS_TGT-gcc                                    \
CXX=$LFS_TGT-g++                                   \
AR=$LFS_TGT-ar                                     \
RANLIB=$LFS_TGT-ranlib                             \
../gcc-4.9.2/configure                             \
    --prefix=/tools                                \
    --with-local-prefix=/tools                     \
    --with-native-system-header-dir=/tools/include \
    --enable-languages=c,c++                       \
    --disable-libstdcxx-pch                        \
    --disable-multilib                             \
    --disable-bootstrap                            \
    --disable-libgomp

make

make install

ln -sv gcc /tools/bin/cc

cd $LFS/sources
rm -rf gcc-build/ gcc-4.9.2/
```
【确认新工具链的基本功能(编译和链接)都是像预期的那样正常工作】
```shell
echo 'main(){}' > dummy.c
cc dummy.c
readelf -l a.out | grep ': /tools'
```
如果一切工作正常的话，这里应该没有错误，最后一个命令的输出形式会是：
[Requesting program interpreter: /tools/lib/ld-linux.so.2]
```shell
rm -v dummy.c a.out
```
【Tcl-8.6.3】
```shell
cd $LFS/sources
tar -xzvf tcl8.6.3-src.tar.gz
cd tcl8.6.3
cd unix
./configure --prefix=/tools
make 
make install
chmod -v u+w /tools/lib/libtcl8.6.so
make install-private-headers
ln -sv tclsh8.6 /tools/bin/tclsh

cd $LFS/sources
rm -rf tcl8.6.3/
```

【Expect-5.45】
```shell
cd $LFS/sources
tar -xzvf expect5.45.tar.gz
cd expect5.45
cp -v configure{,.orig}
sed 's:/usr/local/bin:/bin:' configure.orig > configure
./configure --prefix=/tools       \
            --with-tcl=/tools/lib \
            --with-tclinclude=/tools/include
make
make SCRIPTS="" install

cd $LFS/sources
rm -rf expect5.45/
```
【DejaGNU-1.5.2】
```shell
cd $LFS/sources
tar -xzvf dejagnu-1.5.2.tar.gz
cd dejagnu-1.5.2
./configure --prefix=/tools
make install

cd $LFS/sources
rm -rf dejagnu-1.5.2/
```
【Check-0.9.14】
```shell
cd $LFS/sources
tar -xzvf check-0.9.14.tar.gz
cd check-0.9.14
PKG_CONFIG= ./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf check-0.9.14/
```
【Ncurses-5.9】
```shell
cd $LFS/sources
tar -xzvf ncurses-5.9.tar.gz
cd ncurses-5.9
./configure --prefix=/tools \
            --with-shared   \
            --without-debug \
            --without-ada   \
            --enable-widec  \
            --enable-overwrite
make
make install

cd $LFS/sources
rm -rf ncurses-5.9/
```
【Bash-4.3.30】
```shell
cd $LFS/sources
tar -xzvf bash-4.3.30.tar.gz
cd bash-4.3.30
./configure --prefix=/tools --without-bash-malloc
make
make install
ln -sv bash /tools/bin/sh

cd $LFS/sources
rm -rf bash-4.3.30/
```
【Bzip2-1.0.6】
```shell
cd $LFS/sources
tar -xzvf bzip2-1.0.6.tar.gz
cd bzip2-1.0.6
make
make PREFIX=/tools install

cd $LFS/sources
rm -rf bzip2-1.0.6/
```
【Coreutils-8.23】
```shell
cd $LFS/sources
tar -xvf coreutils-8.23.tar.xz
cd coreutils-8.23
./configure --prefix=/tools --enable-install-program=hostname
make
make install

cd $LFS/sources
rm -rf coreutils-8.23/
```
【Diffutils-3.3】
```shell
cd $LFS/sources
tar -xvf diffutils-3.3.tar.xz
cd diffutils-3.3
./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf diffutils-3.3/
```
【File-5.22】
```shell
cd $LFS/sources
tar -xzvf file-5.22.tar.gz
cd file-5.22
./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf file-5.22/
```
【Findutils-4.4.2】
```shell
cd $LFS/sources
tar -xzvf findutils-4.4.2.tar.gz
cd findutils-4.4.2
./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf findutils-4.4.2/
```
【Gawk-4.1.1】
```shell
cd $LFS/sources
tar -xvf gawk-4.1.1.tar.xz
cd gawk-4.1.1
./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf gawk-4.1.1/
```
【Gettext-0.19.4】
```shell
cd $LFS/sources
tar -xvf gettext-0.19.4.tar.xz
cd gettext-0.19.4
cd gettext-tools
EMACS="no" ./configure --prefix=/tools --disable-shared
make -C gnulib-lib
make -C intl pluralx.c
make -C src msgfmt
make -C src msgmerge
make -C src xgettext
cp -v src/{msgfmt,msgmerge,xgettext} /tools/bin

cd $LFS/sources
rm -rf gettext-0.19.4/
```
【Grep-2.21】
```shell
cd $LFS/sources
tar -xvf grep-2.21.tar.xz
cd grep-2.21
./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf gettext-0.19.4/
```
【Gzip-1.6】
```shell
cd $LFS/sources
tar -xvf gzip-1.6.tar.xz
cd gzip-1.6
./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf gzip-1.6/
```
【M4-1.4.17】
```shell
cd $LFS/sources
tar -xvf m4-1.4.17.tar.xz
cd m4-1.4.17
./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf m4-1.4.17/
```
【Make-4.1】
```shell
cd $LFS/sources
tar -jxvf make-4.1.tar.bz2
cd make-4.1
./configure --prefix=/tools --without-guile
make
make install

cd $LFS/sources
rm -rf m4-1.4.17/
```
【Patch-2.7.4】
```shell
cd $LFS/sources
tar -xvf patch-2.7.4.tar.xz
cd patch-2.7.4
./configure --prefix=/tools
make 
make install

cd $LFS/sources
rm -rf patch-2.7.4/
```
【Perl-5.20.2】
```shell
cd $LFS/sources
tar -jxvf perl-5.20.2.tar.bz2
cd perl-5.20.2
sh Configure -des -Dprefix=/tools -Dlibs=-lm
make
cp -v perl cpan/podlators/pod2man /tools/bin
mkdir -pv /tools/lib/perl5/5.20.2
cp -Rv lib/* /tools/lib/perl5/5.20.2

cd $LFS/sources
rm -rf perl-5.20.2/
```
【Sed-4.2.2】
```shell
cd $LFS/sources
tar -jxvf sed-4.2.2.tar.bz2
cd sed-4.2.2
./configure --prefix=/tools
make
make install

cd $LFS/sources
rm -rf sed-4.2.2/
```
【Tar-1.28】
```shell
cd $LFS/sources
tar -xvf tar-1.28.tar.xz
cd tar-1.28
./configure --prefix=/tools
make 
make install

cd $LFS/sources
rm -rf tar-1.28/
```
【Texinfo-5.2】
```shell
cd $LFS/sources
tar -xvf texinfo-5.2.tar.xz
cd texinfo-5.2
./configure --prefix=/tools
make 
make install

cd $LFS/sources
rm -rf texinfo-5.2/
```
【Util-linux-2.26】
```shell
cd $LFS/sources
tar -xvf util-linux-2.26.tar.xz
cd util-linux-2.26
./configure --prefix=/tools                \
            --without-python               \
            --disable-makeinstall-chown    \
            --without-systemdsystemunitdir \
            PKG_CONFIG=""
make 
make install

cd $LFS/sources
rm -rf util-linux-2.26/
```
【Xz-5.2.0】
```shell
cd $LFS/sources
tar -xvf xz-5.2.0.tar.xz
cd xz-5.2.0
./configure --prefix=/tools
make 
make install

cd $LFS/sources
rm -rf xz-5.2.0/
```
【清理无用内容】
```shell
strip --strip-debug /tools/lib/*
/usr/bin/strip --strip-unneeded /tools/{,s}bin/*

rm -rf /tools/{,share}/{info,man,doc}
```
这两个命令会跳过一些文件，并提示不可识别的文件格式。  
【改变属主】  
＃以后部分的命令都必须以 root 用户身份执行而不再是 lfs 用户。另外，再次确认下 $LFS 变量在 root 用户环境下也有定义。
```shell
su -
echo $LFS
chown -R root:root $LFS/tools
```

##构建 LFS 系统
【准备虚拟内核文件系统】
```shell
mkdir -pv $LFS/{dev,proc,sys,run}

mknod -m 600 $LFS/dev/console c 5 1
mknod -m 666 $LFS/dev/null c 1 3

mount -v --bind /dev $LFS/dev

mount -vt devpts devpts $LFS/dev/pts -o gid=5,mode=620
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run

if [ -h $LFS/dev/shm ]; then
  mkdir -pv $LFS/$(readlink $LFS/dev/shm)
fi
```
【进入 Chroot 环境】
```shell
chroot "$LFS" /tools/bin/env -i \
    HOME=/root                  \
    TERM="$TERM"                \
    PS1='\u:\w\$ '              \
    PATH=/bin:/usr/bin:/sbin:/usr/sbin:/tools/bin \
    /tools/bin/bash --login +h
```
注意一下 bash 的提示符是 I have no name!。这是正常的，因为这个时候 /etc/passwd 文件还没有被创建。  
  
＃从这以后的命令，以及后续章节里的命令都要在 chroot 环境下运行。  
＃如果因为某种原因（比如说重启）离开了这个环境，  
＃请保证运行【准备虚拟内核文件系统】的所有mount命令，再运行本节的命令chroot  
  
【创建目录】
```shell
mkdir -pv /{bin,boot,etc/{opt,sysconfig},home,lib/firmware,mnt,opt}
mkdir -pv /{media/{floppy,cdrom},sbin,srv,var}
install -dv -m 0750 /root
install -dv -m 1777 /tmp /var/tmp
mkdir -pv /usr/{,local/}{bin,include,lib,sbin,src}
mkdir -pv /usr/{,local/}share/{color,dict,doc,info,locale,man}
mkdir -v  /usr/{,local/}share/{misc,terminfo,zoneinfo}
mkdir -v  /usr/libexec
mkdir -pv /usr/{,local/}share/man/man{1..8}

case $(uname -m) in
 x86_64) ln -sv lib /lib64
         ln -sv lib /usr/lib64
         ln -sv lib /usr/local/lib64 ;;
esac

mkdir -v /var/{log,mail,spool}
ln -sv /run /var/run
ln -sv /run/lock /var/lock
mkdir -pv /var/{opt,cache,lib/{color,misc,locate},local}
```
【创建必需的文件和符号链接】
```shell
ln -sv /tools/bin/{bash,cat,echo,pwd,stty} /bin
ln -sv /tools/bin/perl /usr/bin
ln -sv /tools/lib/libgcc_s.so{,.1} /usr/lib
ln -sv /tools/lib/libstdc++.so{,.6} /usr/lib
sed 's/tools/usr/' /tools/lib/libstdc++.la > /usr/lib/libstdc++.la
ln -sv bash /bin/sh

ln -sv /proc/self/mounts /etc/mtab

cat > /etc/passwd << "EOF"
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/dev/null:/bin/false
daemon:x:6:6:Daemon User:/dev/null:/bin/false
messagebus:x:18:18:D-Bus Message Daemon User:/var/run/dbus:/bin/false
systemd-bus-proxy:x:72:72:systemd Bus Proxy:/:/bin/false
systemd-journal-gateway:x:73:73:systemd Journal Gateway:/:/bin/false
systemd-journal-remote:x:74:74:systemd Journal Remote:/:/bin/false
systemd-journal-upload:x:75:75:systemd Journal Upload:/:/bin/false
systemd-network:x:76:76:systemd Network Management:/:/bin/false
systemd-resolve:x:77:77:systemd Resolver:/:/bin/false
systemd-timesync:x:78:78:systemd Time Synchronization:/:/bin/false
nobody:x:99:99:Unprivileged User:/dev/null:/bin/false
EOF

cat > /etc/group << "EOF"
root:x:0:
bin:x:1:daemon
sys:x:2:
kmem:x:3:
tape:x:4:
tty:x:5:
daemon:x:6:
floppy:x:7:
disk:x:8:
lp:x:9:
dialout:x:10:
audio:x:11:
video:x:12:
utmp:x:13:
usb:x:14:
cdrom:x:15:
adm:x:16:
messagebus:x:18:
systemd-journal:x:23:
input:x:24:
mail:x:34:
systemd-bus-proxy:x:72:
systemd-journal-gateway:x:73:
systemd-journal-remote:x:74:
systemd-journal-upload:x:75:
systemd-network:x:76:
systemd-resolve:x:77:
systemd-timesync:x:78:
nogroup:x:99:
users:x:999:
EOF

exec /tools/bin/bash --login +h

touch /var/log/{btmp,lastlog,wtmp}
chgrp -v utmp /var/log/lastlog
chmod -v 664  /var/log/lastlog
chmod -v 600  /var/log/btmp
```



