---
title: Matplotlib无法使用Times new roman字体
date: 2026-01-14 19:28:47
tags:
    - Python
    - Ubuntu
---
<script src="https://fastly.jsdelivr.net/gh/misaka0502/live2d-widget@V0.2/autoload.js"></script>
<!-- <script src="/live2d-widget/autoload.js"></script> -->

解决方案：

```python
sudo apt install msttcorefonts -qq
rm ~/.cache/matplotlib -rf
```