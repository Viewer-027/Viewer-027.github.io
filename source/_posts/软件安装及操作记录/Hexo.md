---
title: HEXO + GitHub CICD 部署个人博客
---

### 参考博客：

[(18 封私信) 使用 GitHub Actions 自动部署 Hexo 博客到 GitHub Pages - 知乎](https://zhuanlan.zhihu.com/p/161969042)

[如何正确的使用 GitHub Actions 实现 Hexo 博客的 CICD | 喵了个咪](https://hdj.me/github-actions-hexo-cicd/#%E5%AF%86%E9%92%A5%E7%94%9F%E6%88%90)

### 下载配置node

```
wget https://nodejs.org/dist/v18.20.3/node-v18.20.3-linux-x64.tar.xz

tar -xvJf node-v18.20.3-linux-x64.tar.xz


ln -s /root/node-v18.20.3-linux-x64/bin/node /usr/local/bin/node
ln -s /root/node-v18.20.3-linux-x64/bin/npm /usr/local/bin/npm

vim /etc/profile

export NODE_HOME=/root/node-v18.20.3-linux-x64/bin/
export PATH=$PATH:$NODE_HOME:/usr/local/bin/

source /etc/profile

node -v
npm -v

echo 'export PATH=$PATH:/root/node-v18.20.3-linux-x64/node_golbal/bin' >> ~/.bashrc

source ~/.bashrc
```
