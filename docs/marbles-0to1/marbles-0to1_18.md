# 4.4 交互演示

## 从零到壹实现 Marbles 资产管理系统 （Fabric-SDK-Node）之－交互演示

应用启动之后，打开浏览器，在地址栏中输入：[`localhost:3000`](http://localhost:3000) 进行访问

首先进入的是应用的初始设置页面，我们可以直接点击 “快速” 按钮，让应用根据启动时指定的配置文件自动创建相关的初始数据。

![1_quick_set](img/7fb5f532e8eaaa0eca7b8830130f1aee.jpg)

初始化完毕之后，应用中有三个用户，每个用户拥有三个 Marble，我们可以对其中的用户进行创建 Marble 资产的操作

![2_create_marble](img/82330874f0bae2e29712057684e68331.jpg)

可以通过单击页面中的 “设置” 按钮，用来启用事务描述模式，

![3_enable_mod](img/e66d1c05bfe0f56fa68dfb7c24ed697e.jpg)

事务描述模式开启之后，就可以看到每个事务详细的执行过程：

![4_create_marble](img/c8d8eeb804a1f0cf199590fc90ff1747.jpg)

可以将指定用户的资产转移给其他用户，直接将 Marble 拖拽到指定用户即可

![4_move_marble](img/bd1ca737994a06a74d958d7fbd7ca5c8.jpg)

也可以将指定用户的 Marble 进行删除，将指定用户拥有的 Marble 拖拽到垃圾箱中即可完成。

![5_del_marble](img/29307da3fc1bd90f2b17ad28fd64dd56.jpg)