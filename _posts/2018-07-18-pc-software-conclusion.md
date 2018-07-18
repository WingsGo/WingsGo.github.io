---
title: 桌面端软件设计经验总结
key: 20180718
tags: 
	- MVC
	- MVVM
---
# 一、MVC #

&nbsp;&nbsp;&nbsp;&nbsp;对于笔者所开发的桌面端软件来说，主要的功能就是通过界面UI与用户进行交互，获取用户参数后将得到的数据在后台进行处理，并将处理后的数据在界面上进行显示。对于这种需求来说，最经典的应用就是MVC架构，可以在图1中粗略展示MVC框架的设计思想。

<!--more-->


![](http://www.ruanyifeng.com/blogimg/asset/2015/bg2015020108.png)



									图1、MVC框架
									
&nbsp;&nbsp;&nbsp;&nbsp;使用这种架构的好处在于可以使得界面与数据分离，使用Controller来控制Model与View间的交互，Controller从View中获取用户数据，并传递给Model进行处理，Model将处理后的结果通过Controller在View上进行显示，达到了数据与界面的分离目的，这样后台可以专注业务逻辑的处理，前端可以专注于界面的优化。
&nbsp;&nbsp;&nbsp;&nbsp;	但使用这种架构也存在一定的问题，那就是	Controller职责过多，当项目越来越大时，Controller中就包含大量的逻辑及Model&View的交互，这就造成了后期项目的维护困难，而且也违背了单一模式的设计原则	
<br/>
				
# 二、开发过程中需要解决的问题 #

    
&nbsp;&nbsp;&nbsp;&nbsp;软件的需求就是实时获取用户的操作，比如查询条件的改变，输入参数的改变，此时我们需要立刻获取用户的输入并实时改变后台的数据及界面显示的结果，如果依旧应用MVC的设计模式，那么我们就要手动更新界面及数据模型，当业务量不复杂的情况下还好，当软件的业务需求逐步增加时，手动更新Model及View无疑会大大增加软件的复杂度，降低软件的可维护性，这个时候，就需要使用另一种更符合业务需求的框架，MVVM。
<br/>

当我们构建好了模型-视图-控制器之间的逻辑关系，由控制器决定好model与view之间的关联关系之后，我们的重心就重模型与视图之间的连接关系中解放出来，着重于处理数据模型的改动，这样只需要改动模型，相应的视图部分也会发生变化，当数据需要再次显示的时候我们只需要将对应的数据传入controlller即可以实时显示视图更新。
<br/>

# 三、MVVM #
&nbsp;&nbsp;&nbsp;&nbsp;MVVM是在MVC基础上衍生出来的一个框架，它的核心在于ViewModel，即**数据模型的双向绑定**，ViewModel的职责在于当View改动时自动更新Model，当Model改变时自动更新View，这样可以简化Controller的职责，我们不需要手动更新Model及View的值，于是我们可以将重心放到业务处理上，大大简化了软件复杂度，提高了软件的可维护性。

![](http://www.ruanyifeng.com/blogimg/asset/2015/bg2015020110.png)
<br/>

# 四、MVC与MVVM的对比 #

&nbsp;&nbsp;&nbsp;&nbsp;接下来我们以一个实例来展示MVC与MVVM的特点以及MVVM的优势所在。我们以一个简单的LineEdit为例，输入框接受用户的输入，并在用户输入结束后添加"LineEdit"作为后缀并显示在LineEdit上。

    class LineEdit:
		def get_string(self):
			return self.string
		
		def set_str(self, value):
			self.string = value
	
	# MVC
	def deal_user_input():
		line_edit = LineEdit()
		string = line_edit.get_string()
		string += "LineEdit"
		line_edit.set_str(string)
	
	# MVVM
	def deal_user_input():
		line_edit = LineEdit()
		line_edit.string.value = line_edit.string.value + "LineEdit"

&nbsp;&nbsp;&nbsp;&nbsp;由上可以看到，使用MVVM实现的LineEdit可以将Model与View进行双向绑定，而避免了Controller手动进行get_string及set_string的更新，关于如何实现一个LineEdit来使用line_edit.string.value进行Model-View的双向绑定可参考Python的三方库[prett库](https://pypi.org/project/prett/)，也可以到[Vue官网](https://cn.vuejs.org/)查看目前最流行的前端框架，理解MVVM的思想，在自己的项目代码中实现。

