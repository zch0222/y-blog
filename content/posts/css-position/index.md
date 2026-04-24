---
date: '2026-04-25T02:41:36+08:00'
draft: true
title: '笔记：CSS `position` 定位总结'
summary: "CSS `position` 定位总结"
description: "CSS `position` 定位总结"

---

## 1. 什么是 `position`

`position` 是 CSS 中用来控制元素定位方式的属性。

常见取值有：

```css
position: static;
position: relative;
position: absolute;
position: fixed;
position: sticky;
```

可以简单理解为：`position` 决定一个元素是按照正常页面布局排列，还是可以通过 `top`、`right`、`bottom`、`left` 进行特殊定位。

---

## 2. 正常文档流是什么

正常文档流可以理解为：页面中的元素按照 HTML 顺序一个接一个排列。

例如：

```html
<div>A</div>
<div>B</div>
<div>C</div>
```

默认显示效果类似：

```text
A
B
C
```

每个元素都会占据自己的位置，后面的元素会排在前面的元素后面。

---

## 3. 什么是“脱离正常文档流”

“脱离正常文档流”指的是：元素不再参与页面正常排队布局，也不再占据原来的位置。

例如：

```html
<div class="a">A</div>
<div class="b">B</div>
<div class="c">C</div>
```

如果 `B` 设置为：

```css
.b {
  position: absolute;
}
```

那么浏览器在排版时会认为 `B` 不占位置，`C` 会往上补位。

可以理解为：

```text
A
C
```

但是 `B` 并不是消失了，它仍然会显示在页面上，只是变成了一个“浮起来”的元素，可能会覆盖其他元素。

一句话总结：

> 正常文档流：元素排队，占座位。  
> 脱离文档流：元素离开队伍，不占座位，但自己还会显示。

---

## 4. `static`：默认定位

```css
.box {
  position: static;
}
```

`static` 是默认值。

特点：

- 元素按照正常文档流排列
- 不支持通过 `top`、`right`、`bottom`、`left` 调整位置
- 页面中大多数元素默认都是 `static`

示例：

```css
.box {
  position: static;
  top: 20px; /* 不生效 */
}
```

可以理解为：

> 该在哪就在哪，不能手动偏移。

---

## 5. `relative`：相对定位

```css
.box {
  position: relative;
  top: 20px;
  left: 10px;
}
```

`relative` 是相对定位。

它是相对于元素原本的位置进行偏移。

特点：

- 不脱离文档流
- 原来的位置仍然保留
- 可以使用 `top`、`right`、`bottom`、`left`
- 常用于给子元素的 `absolute` 定位提供参考

示例：

```css
.box {
  position: relative;
  top: 20px;
}
```

含义：

> 元素视觉上向下移动 20px，但原本的位置还占着。

可以理解为：

```text
A
[B 原来的位置还在]
  B 视觉上向下移动
C
```

---

## 6. `absolute`：绝对定位

```css
.child {
  position: absolute;
  top: 0;
  right: 0;
}
```

`absolute` 是绝对定位。

特点：

- 会脱离正常文档流
- 原来的位置不再保留
- 后面的元素会补位
- 可以使用 `top`、`right`、`bottom`、`left`
- 定位参考是最近的非 `static` 祖先元素

如果没有找到非 `static` 的祖先元素，通常会相对于页面根元素进行定位。

常见写法：

```css
.parent {
  position: relative;
}

.child {
  position: absolute;
  top: 8px;
  right: 8px;
}
```

含义：

> `.child` 相对于 `.parent` 的右上角定位。

这种写法常被称为“父相子绝”。

即：

```text
父元素：position: relative;
子元素：position: absolute;
```

常见用途：

- 角标
- 徽章
- 弹层
- 卡片右上角按钮
- 局部定位元素

---

## 7. `fixed`：固定定位

```css
.box {
  position: fixed;
  top: 100px;
  right: 40px;
}
```

`fixed` 是固定定位。

特点：

- 会脱离正常文档流
- 相对于浏览器窗口定位
- 页面滚动时，元素位置不变
- 可以使用 `top`、`right`、`bottom`、`left`

常见用途：

- 固定导航栏
- 返回顶部按钮
- 右侧悬浮目录
- 悬浮客服按钮

例如右侧固定目录：

```css
.toc {
  position: fixed;
  top: 100px;
  right: 40px;
}
```

含义：

> 目录固定在浏览器右侧，滚动文章时目录不会跟着滚走。

---

## 8. `sticky`：粘性定位

```css
.box {
  position: sticky;
  top: 80px;
}
```

`sticky` 是粘性定位，可以理解为 `relative` 和 `fixed` 的结合。

特点：

- 默认情况下在正常文档流中
- 滚动到指定位置后会“吸住”
- 通常不会像 `absolute` 那样完全脱离布局
- 必须配合 `top`、`bottom` 等属性使用

示例：

```css
.toc {
  position: sticky;
  top: 80px;
}
```

含义：

> 目录原本在页面中正常显示，当页面滚动到距离顶部 80px 时，目录会固定在那里。

常见用途：

- 吸顶导航栏
- 固定表格表头
- 文章目录吸附
- 侧边栏跟随滚动

---

## 9. 五种 `position` 对比

| position | 是否脱离文档流 | 定位参考 | 是否支持 top/right/bottom/left | 常见用途 |
|---|---|---|---|---|
| `static` | 否 | 正常文档流 | 否 | 默认布局 |
| `relative` | 否 | 自己原来的位置 | 是 | 微调位置、作为 absolute 父容器 |
| `absolute` | 是 | 最近的非 static 祖先元素 | 是 | 角标、弹层、局部定位 |
| `fixed` | 是 | 浏览器窗口 | 是 | 悬浮按钮、固定侧边栏 |
| `sticky` | 通常不完全脱离 | 正常位置 + 滚动容器 | 是 | 吸顶、粘性目录 |

---

## 10. `relative` 和 `absolute` 的核心区别

### `relative`

```css
.b {
  position: relative;
  top: 20px;
}
```

特点：

- 不脱离文档流
- 原来的位置还保留
- 只是视觉上发生偏移

可以理解为：

> 人还在队伍里，只是身体偏了一点。

---

### `absolute`

```css
.b {
  position: absolute;
  top: 20px;
}
```

特点：

- 脱离文档流
- 原来的位置不再保留
- 后面的元素会补位
- 元素会浮在页面上

可以理解为：

> 人离开队伍，座位也没了。

---

## 11. `absolute` 和 `fixed` 的区别

### `absolute`

```css
.box {
  position: absolute;
  top: 0;
  right: 0;
}
```

定位参考：

> 最近的非 `static` 祖先元素。

适合做局部定位。

---

### `fixed`

```css
.box {
  position: fixed;
  top: 0;
  right: 0;
}
```

定位参考：

> 浏览器窗口。

适合做全局悬浮定位。

---

## 12. `fixed` 和 `sticky` 的区别

### `fixed`

```css
.box {
  position: fixed;
  top: 80px;
}
```

特点：

- 一开始就固定在窗口某个位置
- 不占原来的布局空间
- 滚动页面时始终不动

---

### `sticky`

```css
.box {
  position: sticky;
  top: 80px;
}
```

特点：

- 一开始在正常文档流中
- 滚动到指定位置后才吸住
- 更适合文章目录、表头、侧边栏

---

## 13. 常见使用场景

### 13.1 父相子绝

```css
.card {
  position: relative;
}

.badge {
  position: absolute;
  top: 8px;
  right: 8px;
}
```

适合：

- 卡片角标
- 右上角关闭按钮
- 图片上的标签
- 局部悬浮元素

---

### 13.2 固定右侧目录

```css
.toc {
  position: fixed;
  top: 100px;
  right: 40px;
  width: 260px;
}
```

适合：

- 博客文章目录
- 文档右侧导航
- 页面悬浮菜单

---

### 13.3 粘性目录

```css
.toc {
  position: sticky;
  top: 80px;
}
```

适合：

- 文章目录随页面滚动
- 滚动到顶部后吸住
- 比 `fixed` 更自然地保留在文章布局中

---

## 14. PaperMod 侧边栏目录示例

在 PaperMod 中，如果想把文章目录固定到右侧，可以使用：

```css
@media (min-width: 1440px) {
  .toc {
    position: fixed;
    top: 100px;
    right: 40px;
    width: 260px;
    max-height: calc(100vh - 140px);
    overflow-y: auto;
    z-index: 10;
  }

  .post-single {
    max-width: 860px;
    margin-left: auto;
    margin-right: auto;
  }
}
```

含义：

- `@media (min-width: 1440px)`：只在大屏幕下生效
- `.toc`：PaperMod 的目录区域
- `position: fixed`：目录固定在浏览器窗口上
- `top: 100px`：距离顶部 100px
- `right: 40px`：距离右侧 40px
- `width: 260px`：目录宽度 260px
- `max-height: calc(100vh - 140px)`：限制目录最大高度
- `overflow-y: auto`：目录太长时内部滚动
- `z-index: 10`：提高层级，避免被其他元素遮挡
- `.post-single`：控制文章正文宽度并居中

---

## 15. 记忆口诀

```text
static：默认排队，不能偏移。
relative：还在队伍，视觉偏移。
absolute：离开队伍，参考父级。
fixed：离开队伍，固定窗口。
sticky：先在队伍，滚动吸住。
```

---

## 16. 简短总结

`position` 的核心是理解两个问题：

1. 元素是否还占据原来的位置？
2. 元素是相对于谁来定位？

其中：

- `static`：默认布局
- `relative`：相对自己原位置移动，不脱离文档流
- `absolute`：脱离文档流，相对最近非 static 父级定位
- `fixed`：脱离文档流，相对浏览器窗口定位
- `sticky`：正常布局和固定定位的结合，滚动到指定位置后吸住

对于博客目录：

- 想一直悬浮在右侧，用 `fixed`
- 想滚动到一定位置后吸住，用 `sticky`