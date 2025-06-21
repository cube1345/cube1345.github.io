[点击查看GitHub仓库](https://github.com/cube1345/react_music)    
本项目是我的第一个使用Ts做的实战项目，使用了React框架，AntD组件库。下面介绍一下我的项目的框架，第一次做的搭建，烦请多多指教~

![Image](https://github.com/user-attachments/assets/68866075-cf21-4c96-96d3-0d8cbca8d78f)  

# 如图所示，这是在src目录下的第一层子目录：

assets文件夹用来存储静态资源；  
component文件夹用于存储可复用组件；  
page文件夹用于存储主要页面；  
router文件夹用于管理路由信息；

## assets文件夹管理：

![Image](https://github.com/user-attachments/assets/1444720b-6497-4dfb-a4c9-4690f7a80c60)

css存储初始css配置以及通用css样式；
date存储静态数据资源；
img存储静态图片资源；

![Image](https://github.com/user-attachments/assets/39fa987d-d57f-48fc-b3f1-b01c5384d8e6)

## page文件夹下面的子文件夹将对应本项目的以及路由：

### 本项目最复杂的page

![Image](https://github.com/user-attachments/assets/5aa13cb2-86fc-422e-b57b-a840bd7e7d4d)

child-cpn 对应该page下可复用组件；
child-view对应该page下页面，在本项目中分别对应各个二级路由；

#### page下的子page

![Image](https://github.com/user-attachments/assets/4a7cb1db-e5bb-44f4-849e-d77707a0c622)

以recommand为例，此处将与recommand相关的网络请求统一放置于此，使得网络请求的分类更加明显。



没怎么写过东西，实在没话讲~