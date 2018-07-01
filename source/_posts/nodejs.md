---
title: 安装Node.js
date: 2017-10-28
tags: Node.js
---

### Linux

```shell
curl -sL https://deb.nodesource.com/setup_6.x | sudo -E bash -;
sudo apt-get install -y nodejs;
```

### Mac OSX

```shell
curl "https://nodejs.org/dist/latest/node-${VERSION:-$(wget -qO- https://nodejs.org/dist/latest/ | sed -nE 's|.*>node-(.*)\.pkg</a>.*|\1|p')}.pkg" > "$HOME/Downloads/node-latest.pkg" && sudo installer -store -pkg "$HOME/Downloads/node-latest.pkg" -target "/";
brew install node;
```

### 参考

- <https://nodejs.org/en/download/package-manager/>

