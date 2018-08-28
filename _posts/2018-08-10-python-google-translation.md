---
layout: post
title: Python应用-判断单词-合并换行-自动Google翻译文献
---
以下我们选取TI公司搭载DaVinci Video Processors的DM814x/AM387x EVM Baseboard为平台来讨论嵌入式系统、交叉编译环境的一些常识。

#  1、背景 #
研究生要写出好一点的论文，必然是要看许多的文献。当我们看外文文献，Google翻译就是个好帮手。即便如此，这其中依然少不了折腾。
通常我们看到的论文是这样的：
![image01](https://raw.githubusercontent.com/liuqiaoping7/liuqiaoping7.github.io/master/images/img01.jpg)
复制出来的文字粘贴到Google翻译结果是这样的：
![image02](https://raw.githubusercontent.com/liuqiaoping7/liuqiaoping7.github.io/master/images/img02.jpg)

#  2、循序渐进 #
##    2.1 原始 #
问题罪魁祸首是PDF论文复制出来的换行符号。这还不简单，逐个删除换行符号就行了？
![image03](https://raw.githubusercontent.com/liuqiaoping7/liuqiaoping7.github.io/master/images/img03.jpg)
可以看到翻译结果明显准确了！可是重复的事情做多了，大概你也就烦了。
##    2.2 石器 #
重复性的工作嘛，机器是最乐意的了。最容易想到的是：复制到一个文本编辑器里，查找全局替换就好了。
以atom编辑器为例：
先要知道替换对象是什么，我们设置atom显示空白字符：
![image04](https://raw.githubusercontent.com/liuqiaoping7/liuqiaoping7.github.io/master/images/img04.jpg)
罪魁祸首显形了\r\n：
![image05](https://raw.githubusercontent.com/liuqiaoping7/liuqiaoping7.github.io/master/images/img05.jpg)
使用全局替换，这里注意必须使用**正则表达式**才能替换：
![image06](https://raw.githubusercontent.com/liuqiaoping7/liuqiaoping7.github.io/master/images/img06.jpg)
##    2.3 铁器 #
接来下我们来个跳跃，自己生产自己的工具，让更好的工具来替我们*替换*好了，步骤如下：
+我们从PDF按下Ctrl+C
+工具替换粘贴板文本里的\r\n
+我们粘贴到Google翻译
我们的这个工具可以是Python脚本，由他定时监视我们的粘贴板：
```python
import pyperclip
import time

def altercopy():
    tempBuff=' '    #仅仅用于暂存判定
    while True:
        time.sleep(3)
        pasteText=pyperclip.paste()

        if tempBuff != pasteText:
            tempBuff=pasteText

            strBuff=pasteText
            strBuff = strBuff.replace('\r\n', ' ')

            pyperclip.copy(strBuff)    #修改粘贴板
            tempBuff=strBuff    #防止循环
```
于是乎我们省心了不少。
##    2.4 蒸汽 #
使用过林格斯词典大多体验过这个功能：选中内容后词典会自动弹出翻译结果来。我们要求不高，当我复制了内容之后是否可以为我们弹出Google翻译结果呢？
模拟步骤如下：
+我们从PDF按下Ctrl+C
+工具替换粘贴板文本里的\r\n
+工具打开浏览器，访问Goolge翻译网页翻译粘贴板文本
这里我们需要了解一下url的常识，不过我们暂且不深究。在浏览器上翻译hello看看就明白了。直观的就是 https://translate.google.com/?hl=zh-CN&tab=wT#en/zh-CN/ + 'hello'。也就是访问[hello](https://translate.google.com/?hl=zh-CN&tab=wT#en/zh-CN/hello)
![image07](https://raw.githubusercontent.com/liuqiaoping7/liuqiaoping7.github.io/master/images/img07.jpg)
```python
import pyperclip
import time

def translation():
    tempBuff=' '    #仅仅用于暂存判定
    while True:
        time.sleep(3)
        pasteText=pyperclip.paste()

        if tempBuff != pasteText:
            tempBuff=pasteText

            strBuff=pasteText
            strBuff = strBuff.replace('\r\n', ' ')

            url='https://translate.google.com/?hl=zh-CN&tab=wT#en/zh-CN/'+strBuff
            chrome_path=r"C:\Program Files (x86)\Google\Chrome\Application\chrome.exe"    #r代表不转义，否则\需要替换为\\
            webbrowser.register('chrome', None,webbrowser.BackgroundBrowser(chrome_path))
            webbrowser.get('chrome').open(url)
```
咋一试，真的是省心哈！都说出来混迟早要还呢。当复制下面一段：
> In this work, we demonstrate a hierarchical
classification tree that filters and classifies a received signal
as AM, FM, 4/16/64-QAM, 2/4/8-PAM, 4/8/16-PSK, DSSS, and
FSK. Coarse estimates of signal parameters are obtained from
energy detection and are refined using cyclostationary estimators.

意外的得到了：
![image08](https://raw.githubusercontent.com/liuqiaoping7/liuqiaoping7.github.io/master/images/img08.jpg)
明眼人一看就明白了翻译内容 '/' 和后面的内容都没有出现在Google翻译框中！这其实就是因为url的规则。多的不啰嗦，大家伙看看url编码就知道了。我们解决这个问题就需要**特殊字符转义编码** ：
- +     %2B
- ?     %3F
- %    %25
- #     %23
- &    %26
这里注意转义编码需要避免**重复转义**，在这里就是'%'需要先转，代码如下：
```python
import pyperclip
import time

def translation():
    tempBuff=' '    #仅仅用于暂存判定
    while True:
        time.sleep(3)
        pasteText=pyperclip.paste()

        if tempBuff != pasteText:
            tempBuff=pasteText

            strBuff=pasteText
            strBuff = strBuff.replace('\r\n', ' ')

            strBuff = strBuff.replace('%', '%25')    #url转义 %一定要在最前面
            strBuff = strBuff.replace('+ ', ' %2B')
            strBuff = strBuff.replace('/', '%2F')
            strBuff = strBuff.replace('?', '%3F')
            strBuff = strBuff.replace('#', '%23')
            strBuff = strBuff.replace('&', '%26')
            url='https://translate.google.com/?hl=zh-CN&tab=wT#en/zh-CN/'+strBuff
            chrome_path=r"C:\Program Files (x86)\Google\Chrome\Application\chrome.exe"    #r代表不转义，否则\需要替换为\\
            webbrowser.register('chrome', None,webbrowser.BackgroundBrowser(chrome_path))
            webbrowser.get('chrome').open(url)
```
##    2.5 电气自动 #
以上我们只是机械地把*换行*替换为*空格*，有时候不仅在词间换行，也会有断字换行的情况，这个时候正确的处理应该是把*换行*替换为*空字符*。这里最核心的需求是判断是否断字，等价于判断单词是否有效。这里我们大材小用一下Natural language toolkit (NLTK)。
```python
import pyperclip
import time
import nltk
from nltk.corpus import words as words_range

def translation():
    tempBuff=' '    #仅仅用于暂存判定
    while True:
        time.sleep(3)
        pasteText=pyperclip.paste()

        if tempBuff != pasteText:
            tempBuff=pasteText

            strBuff=pasteText
            while  strBuff.find('\r\n') != -1:
                lines= strBuff.split('\r\n',1)
                line=lines[0]
                words=line.rsplit(' ',1)
                lastword=words[-1].lower()
                if (lastword in words_range.words() or lastword.rstrip('s') in words_range.words() or lastword.rstrip('es') in words_range.words()):
                    strBuff =  strBuff.replace('\r\n', ' ',1)    #正常换行
                else :
                    print(lastword)
                    strBuff =  strBuff.replace('\r\n', '',1)    #断词换行

            strBuff = strBuff.replace('%', '%25')    #url转义 %一定要在最前面
            strBuff = strBuff.replace('+ ', ' %2B')
            strBuff = strBuff.replace('/', '%2F')
            strBuff = strBuff.replace('?', '%3F')
            strBuff = strBuff.replace('#', '%23')
            strBuff = strBuff.replace('&', '%26')
            url='https://translate.google.com/?hl=zh-CN&tab=wT#en/zh-CN/'+strBuff
            chrome_path=r"C:\Program Files (x86)\Google\Chrome\Application\chrome.exe"    #r代表不转义，否则\需要替换为\\
            webbrowser.register('chrome', None,webbrowser.BackgroundBrowser(chrome_path))
            webbrowser.get('chrome').open(url)
```
运行示例：
选择Ctrl+C:
![image09](https://raw.githubusercontent.com/liuqiaoping7/liuqiaoping7.github.io/master/images/img09.jpg)
启动脚本和输出：
![image10](https://raw.githubusercontent.com/liuqiaoping7/liuqiaoping7.github.io/master/images/img10.jpg)
自动打开网页翻译内容：
![image11](https://raw.githubusercontent.com/liuqiaoping7/liuqiaoping7.github.io/master/images/img11.jpg)
#  3、感想 #
Python这门工具语言，对于字符串等序列操作极其简明，各方面支持库也非常丰富。日常数据处理极其便利，对于不同场景需求应该多加应用！
