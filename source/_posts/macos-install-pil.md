---
title: MacOS安装PIL
date: 2017-12-04
tags: [MacOS]
toc: true
---

MacOS安装PIL库

安装libjpeg

```shell
curl -O http://www.ijg.org/files/jpegsrc.v8c.tar.gz
tar -xvzf jpegsrc.v8c.tar.gz
cd jpeg-8c
./configure
make
sudo make install
cd ../
```

安装freetype

```shell
curl -O http://ftp.igh.cnrs.fr/pub/nongnu/freetype/freetype-2.4.5.tar.gz
tar -xvzf freetype-2.4.5.tar.gz
cd freetype-2.4.5
./configure
make
sudo make install
cd ../
```

安装PIL

```
curl -O -L http://effbot.org/media/downloads/Imaging-1.1.7.tar.gz
# extract
tar -xzf Imaging-1.1.7.tar.gz
cd Imaging-1.1.7
python setup.py install
cd ..
```

### 参考

- [stackoverflow install pil on mac OSX](https://stackoverflow.com/questions/9070074/how-can-i-install-pil-on-mac-os-x-10-7-2-lion)