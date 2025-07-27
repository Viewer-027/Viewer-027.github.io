HEXO + GitLab 部署个人博客

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
