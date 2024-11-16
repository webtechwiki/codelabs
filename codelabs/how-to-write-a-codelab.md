summary: 如何编写 Codelab
id: how-to-write-a-codelab
categories: codelab
tags: codelab
status: Published
authors: panhy

# 如何编写 Codelab

## 概述  

Duration: 1

### 你将学到什么  

- 如何设置每个幻灯片所需的时间  
- 如何包含代码片段  
- 如何超链接项目  
- 如何插入图片  
- 其他技巧  

---

## 设置时长  

Duration: 2

要指示浏览每张幻灯片所需的时间，请在每个二级标题（即 `##`）下设置 `Duration` 为一个整数。  
这个整数代表分钟数。例如，设置 `Duration: 4` 表示完成这一张幻灯片需要 4 分钟。  

总时间将会自动计算，并在你创建 Codelab 后显示出来。  

---

## 代码片段  

Duration: 3

要包含代码片段，你可以采用以下方法：  

- **内联高亮**：使用键盘上的反引号（`` ` ``）来实现。  
- **嵌入代码**：使用代码块显示完整代码。  

### JavaScript 示例  

```javascript
{ 
  key1: "string", 
  key2: integer,
  key3: "string"
}
```

### Java 示例  

```java
for (statement 1; statement 2; statement 3) {
  // 要执行的代码块
}
```

---

## 超链接和嵌入图片  

Duration: 1

### 超链接  

[YouTube - Halsey 播放列表](https://www.youtube.com/user/iamhalsey/playlists)  

### 图片  

![图片替代文本](assets/puppy.jpg)  

---

## 其他技巧  

Duration: 1

查看官方文档：[Codelab 格式指南](https://github.com/googlecodelabs/tools/blob/master/FORMAT-GUIDE.md)  
