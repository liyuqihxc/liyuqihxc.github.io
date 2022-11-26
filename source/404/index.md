---
title: '404 - 真巧，竟然在这里遇到你！'
toc: false
comments: false
permalink: /404.html
---

<!-- markdownlint-disable MD039 MD033 -->

## 这是一个不存在的页面

很抱歉，你想要找的页面并不存在。

即将在 <span id="timeout">5</span> 秒后返回首页。

你也可以 **[点击这里](/)** 立即返回首页。

<script>
let countTime = 5;

function count() {
  
  document.getElementById('timeout').textContent = countTime;
  countTime -= 1;
  if(countTime === 0){
    location.href = '/';
  }
  setTimeout(() => {
    count();
  }, 1000);
}

count();
</script>