# [玩具 Idea] 影子格斗


今天看小宁子介绍 '拓麻歌子' 宠物蛋时想到了一个游戏机的玩法！大致思路是，在房间中心扫描某一面墙，获取深度信息，将墙上突出来的部分当作 **平台**，利用投影仪技术将 角色/影子 投影到墙上。在 2D 的墙面上进行 战斗和跳跃。玩家可以自行在墙上添加物体！

1. 一个**深度摄像头**
2. 一个**激光投影仪**

难点是深度摄像头和激光投影仪的**对齐问题**， 还有成本问题！

好想要做一个这样的东西，一定非常有趣！！

这就像小时候玩过的手影，还会和小伙伴用手影战斗！！！非常有意思。

深度摄像头其实已经很便宜了，不管是 iphone 还是kinect 都可以，主要是激光投影仪太贵了，而且不是家中常有的东西。并且一般的投影仪并不能在图像透明通道不发光，如果是全发光的话效果并不好。

等到深度传感器技术和激光投影技术成本更低廉的时候再来实现这个想法！！


可以考虑 **便携式迷你投影仪**
1. AAXA P2-B - 130 Lumen Mini Projector (这个非常适合s)
2. Kodak 柯达超小型便携式投影仪
对于深度摄像机也完全没有必要这么严格，只需要**普通相机**，对画面进行**边缘检测**即可！


暂时不用投影仪来做，可以先实现一个简单功能，就是 在白纸上画一些平台和障碍物，或者贴纸。用相机拍下来，然后游戏自动生成这个场景，然后就能玩了！
