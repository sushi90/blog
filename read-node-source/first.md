#Makefile
首先通过Makefile来看nodejs的构建过程。
Make 命令如果不加任何的参数，会默认执行第一个target

-include config.mk 载入Make时候的配置文件，这个文件是./configure的时候可以配置和生成的。-include表示即使加载config.mk错误也不理会。

BUILDTYPE ?= Release ?=表示如果这儿变量为空，则声明它。其实，相当于是config.mk的默认值。
OSTYPE := $(shell uname -s | tr '[A-Z]' '[a-z]') :=定义的时候扩展，就是说不按运行时改变。

ifeq ($(BUILDTYPE),Release)  判断来make那个版本
all: out/Makefile $(NODE_EXE)
else
all: out/Makefile $(NODE_EXE) $(NODE_G_EXE)
endif

$(PYTHON) tools/gyp_node.py -f make  首先用python 运行 tools/gyp_node.py。这个文件的目的是什么？

  if [ -f $@ ]; then   []相当于test。-f表示这个文件存在，并且是一个regular file
		$(error Stale $@, please re-run ./configure)
	else
		$(error No $@, please run ./configure first)
	fi

  顶上这些都是确认文件的有效性的。接下来就是要安装了
  make install 
  