#find nodejs entrance
通过Makefile我们可以发现node是有gyp来构建的。找到node.gyp文件，在其中的target->conditions->sources中找到入口文件src/node_main.cc