---
layout: post_layout
title: Sublime Text 3 中文乱码解决
time: 2019年01月09日 星期四
location: 杭州
pulished: true
excerpt_separator: "Step1. 快捷键 **Ctrl"
---

### Sublime Text 3 中文乱码解决

Step1. 快捷键 **Ctrl + ~** 打开命令行，输入：`import urllib.request,os,hashlib; h = '6f4c264a24d933ce70df5dedcf1dcaeeebe013ee18cced0ef93d5f746d80ef60'; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); by = urllib.request.urlopen( 'http://sublime.wbond.net/' + pf.replace(' ', '%20')).read(); dh = hashlib.sha256(by).hexdigest(); print('Error validating download (got %s instead of %s), please try manual install' % (dh, h)) if dh != h else open(os.path.join( ipp, pf), 'wb' ).write(by)`

Step2. 快捷键 **Ctrl + Shift + P** 在弹框中输入 **Package Control：install Package**。稍等，之后在弹框中搜索 **ConvertToUTF8** 安装。完成后重启即可。