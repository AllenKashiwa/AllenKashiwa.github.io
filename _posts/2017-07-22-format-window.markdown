---
layout:     post
title:      "Unity格式化代码工具"
date:       2017-07-22 14:50:00
---

# 文本编码与换行符

程序员都知道为了让只能处理数字的计算机处理文本，计算机科学家们将文本与数字对应形成编码，如著名的**ASCII**和**UTF8**。由于历史与地域的原因，目前的编码已经非常丰富。于此同时，文本中的换行符不像编码那么多种多样但常用的也不只一种，到底用**CRLF**还是**LF**就得好好考虑一下。

在同一个项目中的不同文件可能采用了不同的编码和换行符。你当然可以要求团队成员新建文件时使用何种编码以及何种换行符，但当你使用第三方库时总不能保证其他人使用的编码与你的一致。为了不踩进编码和换行符的坑中，咱们不如搞个自动替换工具，一劳永逸解决这个问题。

# 搞个工具

Unity编辑器本身是很容易扩展的，我写了这样子的一个小工具来帮助我统一编码和换行符还有到底是用空哥还是制表符来当缩进的千古难题：

![image](http://baizihan.me/assets/images/in-post/format_window/01.png)

而这样的一个工具核心功能的实现也很简单：

```
private void HandleCurFile(string fileName)
{
    try
    {
        string content = File.ReadAllText(fileName);

        // 替换换行符
        content = content.Replace("\r", "");
        content = content.Replace("\n", lineEndings[selectLineEndingIndex]);

        // 处理制表符
        content = isInsertSpaces ? content.Replace("\t", new string(' ', spaceCount)) : content.Replace(new string(' ', spaceCount), "\t");

        // 按对应编码写入文件
        File.WriteAllText(fileName, content, encodings[selectEncodingIndex]);
    }
    catch (Exception ex)
    {
        Debug.LogErrorFormat("FormatWindow HandleCurFile Format file faild, fileName: {0} msg: {1}", fileName, ex.Message);
    }
}
```

要查看完整代码，可以前往我的[github](https://github.com/AllenKashiwa/StudyUnity)中的FormatCode这个项目。

# 支持
最后，如果你喜欢本文，欢迎进行关注，打赏，点赞，点Star。