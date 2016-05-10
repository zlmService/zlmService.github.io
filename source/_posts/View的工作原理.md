---
title: View的工作原理
date: 2016-05-09 21:23:53
tags: 源自Android开发艺术探索总结
---

### MeasureSpec：测量规格
- 转换过程：对于**DecorView**（顶级View）来说，MeasureSpec由窗口的尺寸和自身的LayoutParams来共同确定的；对于**普通的View**，其MeasureSpec由父容器的MeasureSpec和自身的LayoutParams来共同确定的。
- **SpecMode**：测量模式
	1. UNSPECIFIED：父容器不做任何限制，要多大给多大，一般用于系统内部
	2. EXACTLY：父容器已经检测出View所需要的精确大小，相当于match_parent
	3. AT__MOST：父容器指定了一个可用大小即SpecSize，相当于wrap_content
- **SpecSize**：某种测量模式下的规格大小

![](.//viewMeasureSpec.png)
