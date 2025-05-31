看了keysking的最新视频，恰好Jetbrains的Clion软件最近对非商业用途免费开放，所以心血来潮自己配置了一下，之后便可以不再使用CubeIDE了。
下面是参考原视频的操作步骤：
### 1.下载Clion，CubeMX，CubeCLT
Clion的下载没什么好说的，非常简单。CubeMX在使用Clion时，将代替CubeIDE的作用，作为给STM32的图形化配置。但无论是CubeMX还是CubeCLT在ST的官网上比较难下的，（ST的官网是真的卡。。。好不容易登进去了还会报404，我在之前使用CubeMX的时候真的费了很大的劲），不过keysking的FubeMX出来之后还是很好用的，各种固件包，包括CubeMX，CubeIDE,CubeCLT都可以在这里下载；另外就是ST的账号注册问题，我去年注册费了很大劲。值得注意的点主要是，我当时使用的邮箱是网易邮箱，@163.com，QQ邮箱尝试好几次未能成功。
       关于FubeMX下载值得注意的是，可能会出现以下这个情况：

![Image](https://github.com/user-attachments/assets/19f3f1d4-4501-44ee-b0e6-e2e5d2251416)
这个不是难题，点击图片上的三个点，然后保留>>显示详细信息>>仍然保留即可解决。使用FubeMX在下载这些东西就很方便了。
### 2.配置Clion

1. 第一步

Clion我这里是直接导入了之前使用的IDEA的配置。关于STM32单片机开发，首先要把你的CubeMX和CubeCLT的位置告诉Clion，这个在一开始弹出的界面就可以配置好，至于OpenOCD可以先不管。此时，可以先去打开CubeMX进行图形化配置（需要注意在Project Manager中选择CMake），并将生成的文件夹导入到Clion中。接下来主要需要注意的地方如图所示，上面的名称我们在
下一步要用。下面四个全部在刚才下载的CubeCLT中都能找到，并且一般是自动找到，不需要手动找这些可执行文件。

![Image](https://github.com/user-attachments/assets/3881ccad-6948-4556-a02e-8c95d6175c82)

  2.第二步

第二个需要取配置的地方是这里，这里可以使用Debug模式，工具链栏需要写我们刚才那个界面取的名称，另外，“在编辑CMakeLists.txt或其他CMake配置文件时重新加载CMake项目”这个也需要打勾。
![Image](https://github.com/user-attachments/assets/e119acff-c2a7-4780-a85e-6bcbdcb0dc1d)

3.第三步
在高级设置中，（一个比较靠后的位置，建议从后往前找），在“调试器一栏”最后一项，启用服务调试器，选上。
![Image](https://github.com/user-attachments/assets/093353e7-176b-44a6-a3fa-02c5843d88be)
选上之后就会在最上方出现这个

![Image](https://github.com/user-attachments/assets/abf92f90-9fb4-4600-a50f-01f04b397301)
选择“原生”下面的“编辑调试服务器”，在弹出的窗口下选择ST-Link。

![Image](https://github.com/user-attachments/assets/a54426f6-dbee-470f-b807-2089178ac794)


至此，Clion的配置结束，可以正常的使用Clion进行STM32开发了。Clion强大的代码补全、美观的UI设计，以及丰富的插件市场，都是开发STM32的利器。




