---
layout: post
title:  "Maven 私库"
date:   2016-05-12 09:00:00 +0800
categories: 基础设施搭建
---

### Maven 介绍

### 机器准备
10.10.109.162 10.10.111.218

### Setup

{% highlight shell %}

mkdir -p /data/webapps/
cd /data/webapps

wget https://sonatype-download.global.ssl.fastly.net/nexus/3/nexus-3.0.0-03-unix.tar.gz
tar zxvf nexus-3.0.0-03-unix.tar.gz
ln -s /data/webapps/nexus-3.0.0-03 nexus
rm -f nexus-3.0.0-03-unix.tar.gz

chown nobody:nobody /data/webapps -R
sudo -u nobody nohup bin/nexus start

{% endhighlight %}