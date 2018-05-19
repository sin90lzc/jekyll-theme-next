关于Mac的一些使用经验都在这里进行汇总了。

### bash启动脚本

> 参考[Bash startup files in macOS](https://ss64.com/osx/syntax-bashrc.html)

如果bash以login shell的形式启动，启动脚本的读取顺序是/etc/profile -> ~/.bash_profile -> ~/.bash_login -> ~/.profile

如果bash不以login shell的形式启动，启动脚本读取的是~/.bashrc

因此，最佳实践是在~/.bash_profile中添加如下内容：

	if [ -f ~/.bashrc ]; then
		source ~/.bashrc
	fi
	
在.bashrc中添加如下内容：

	if [ -f ~/.bash-options ]; then
		. ~/.bash-options
	fi
	if [ -f ~/.bash-aliases ]; then
		source ~/.bash-aliases 
	fi
	if [ -f ~/.bash-functions ]; then
		source ~/.bash-functions
	fi
	
~/.bash-options,~/.bash-aliases,~/.bash-functions则按分类写脚本内容。


