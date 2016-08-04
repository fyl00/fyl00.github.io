---
layout:     post
title:      一次用 Python 写 GUI 的经历
date:       2016-08-04
summary:    之前写了个脚本自己平常工作用，最近因为需要把脚本给其他同事用，为了避免它们还需要配置环境，于是用 PyQt 写了个界面，然后用 PyInstaller 打包成 exe 了，过程中遇到了一些坑，纪录一下。
categories: python
---

> 之前写了个脚本自己平常工作用，最近因为需要把脚本给其他同事用，为了避免它们还需要配置环境，于是用 PyQt 写了个界面，然后用 PyInstaller 打包成 exe 了，过程中遇到了一些坑，纪录一下。

### PyQt

之前写过一个[虾米歌单导出](http://xiamilist.sinaapp.com/)小脚本，exe 版本是用的 Python 自带的 Tkinter 写的，所以这次尝试用 PyQt。写的时候遇到了任务栏闪烁和打包后程序界面图标不显示的问题。

- 任务栏闪烁提示

  后台运行程序的时候，如果使用 `QFileDialog` ，程序弹出保存对话框的时候会在任务栏闪烁提示。但不是每个 Widget 都会有这个功能，但是这时候可以通过 `QtWidget` 的 [activateWindow()](http://doc.qt.io/qt-4.8/qwidget.html#activateWindow) 和 [QApplication.alert()](http://doc.qt.io/qt-5/qapplication.html#alert) 来实现，但这两者使用的时候会有点区别。

  `activateWindow()` 是实例方法，只要程序窗口切换到前台被激活了就不会再闪烁。而 `QApplication.alert(QWidget, msecs: int = 0)` 是静态方法，`msecs` 值为 0 的时候，和 `activateWindow()` 的效果一致，但是需要在闪烁停了之后激活程序才会任务栏窗口颜色才会恢复正常，否则还是黄色，直到闪烁停了之后再激活一次程序。`msecs` 不为 0 的时候，超过这个时间后，任务栏窗口就会自动恢复正常。**所以 `activateWindow()` 是比较好的选择，除非不要求一直在任务栏提示。**

- 资源文件在打包时的处理

  把代码用 PyInstaller 打包到一个文件的时候，会出现图标或图片都不显示的情况。这种时候就需要用 Qt’s resource system 来对其进行打包。

  首先新建一个 .qrc 文件，内容格式如下：

  ```xml
  <RCC>
    <qresource prefix="/" >
      <file>img/image1.png</file>
      <file>img/image2.png</file>
      <file>img/image3.png</file>
    </qresource>
  </RCC>
  ```

  然后去文件目录下执行 `pyrcc5 -o images_qr.py images.qrc  ` 命令，最后在代码中 import image_qr.py，并且修改下图片路径，一定要在路径前面加上冒号。

  ```python
  import image_qr.py
  # your code
  self.setWindowIcon(QtGui.QIcon(':/img/image1.png'))
  ```

  ​

### PyInstaller

之前用 cx_freeze 打过包，但感觉不理想，搜索了下发现 PyInstaller 用得比较多，就使用了这个，第一次用的时候没遇到问题，**但是因为 Python 是装的 64 位的，所以打包之后的程序无法在 32 位的机器上使用**。这就意味着我必须要在 Python 32 位的环境下打包，但重新配置了下环境之后，遇到了 ImportError 和无法链接到动态库的问题。

- ImportError: DLL load failed

  用 PyInstaller 给程序打包的时候遇到了`pyi_rth_qt5plugins returned -1` 的 Fatal Error 提醒。这个错误信息几乎是毫无用处的，修改 .spec 文件，打开 debug 模式以及显示 console 后，在 `running pyi_rth_qt5plugins.py` 的时候发生了 `ImportError: DLL load failed 找不到指定的模块` 错误。

  回头想起了在编译的时候看到了很多 **WARNNING** 消息，回过头查看，发现了很多 `lib not found` 的问题。但是仔细检查了 Python 库之后，这些 DLL 明明在 `C:\Python35-32\Lib\site-packages\PyQt5\Qt\bin` 目录下，最后的搜索得到的解决办法是把这个目录添加到环境变量里。

  问题得到了解决，猜测原因是因为之前使用 pip 安装 PyQt 的时候， pypi.python.org 总是连接不顺畅，最后去下了个 .whl 文件直接安装，导致没有对应环境变量打包的时候找不到 DLL。

  ```shell
  # 环境： Python == 3.5.2 ，PyInstaller == 3.1.1

  10951 WARNING: lib not found: Qt5Svg.dll dependency of C:\python35-32\lib\site-packages\PyQt5\Qt\plugins\imageformats\qsvg.dll
  11206 WARNING: lib not found: Qt5Gui.dll dependency of C:\python35-32\lib\site-packages\PyQt5\Qt\plugins\imageformats\qsvg.dll
  11437 WARNING: lib not found: Qt5Core.dll dependency of C:\python35-32\lib\site-packages\PyQt5\Qt\plugins\imageformats\qsvg.dll
  11763 WARNING: lib not found: Qt5Gui.dll dependency of C:\python35-32\lib\site-packages\PyQt5\Qt\plugins\imageformats\qtga.dll
  12017 WARNING: lib not found: Qt5Core.dll dependency of C:\python35-32\lib\site-packages\PyQt5\Qt\plugins\imageformats\qtga.dll
  12224 WARNING: lib not found: Qt5Gui.dll dependency of C:\python35-32\lib\site-packages\PyQt5\Qt\plugins\platforms\qminimal.dll
  12418 WARNING: lib not found: Qt5Core.dll dependency of C:\python35-32\lib\site-packages\PyQt5\Qt\plugins\platforms\qminimal.dll
  12625 WARNING: lib not found: Qt5Gui.dll dependency of C:\python35-32\lib\site-packages\PyQt5\Qt\plugins\platforms\qwindows.dll
  12833 WARNING: lib not found: Qt5Core.dll dependency of C:\python35-32\lib\site-packages\PyQt5\Qt\plugins\platforms\qwindows.dll
  ```

- api-ms-win-crt-runtime 错误

  PyInstaller 打包之后的程序运行的时候发生 `api-ms-win-crt-runtime` 动态库之类的错误，似乎只有在 Python 3.5 下打包才会遇到。因为 Universal CRT （KB2999226）缺失，可以通过安装此更新来解决问题。或者直接下载 Visual C++ Redistributable ([x86](http://download.microsoft.com/download/9/3/F/93FCF1E7-E6A4-478B-96E7-D4B285925B00/vc_redist.x86.exe) ,[x64](http://download.microsoft.com/download/9/3/F/93FCF1E7-E6A4-478B-96E7-D4B285925B00/vc_redist.x64.exe) )。

  参考链接：[api-ms-win-crt-runtime-l1-1-0.dll is missing when open office file](http://stackoverflow.com/questions/33265663/api-ms-win-crt-runtime-l1-1-0-dll-is-missing-when-open-office-file) 

这次的几个坑让我更坚定这类工具最好不要用最新版本，一不见得更稳定，二是遇到问题资料也比较少，在这类不必要的麻烦上花太多时间不值得，除非新版本能有很大的收益。

