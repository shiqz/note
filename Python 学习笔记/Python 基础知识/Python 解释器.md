Python 解释器就是用来执行 Python 语言编写的脚本程序的，安装好Python后，它的解释器默认在 `/usr/local/bin/python3.10`，我们可以将它加入到系统的环境变量中去，这样我们解可以在终端的任何地方使用它了。
> 如果你安装的其他位置，请自己注意查看了。

如何进入Python解释器，我们只需要在终端中命令行中输入：`python` 后回车即可进入，我们输出一个 `Hello world` 看看效果吧：
![运行效果图](https://api.shiqz.cn/static/upload/20220630/1656571935191439.png)
`Command + D` 快捷键可以退出解释器，或者输入 `quit()` 也可以退出。

---

当然，我们也可以不用进入解释器也可以执行相关语句，例如：
```bash
python -c "print('Hello')"

# Hello
```
交互模式下需要加上 `-i` 参数执行。