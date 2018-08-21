只考虑主流设备的话，iOS的屏幕适配其实已经相对简单了。但如果考虑上老的设备，如iPhone  4/5，适配的繁杂度还是蛮高。这里介绍一种简单的适配方案，在我看来，适配效果还是蛮不错的。

考虑设计师输出的产品设计图，普遍是基于iPhone 6即4.7英寸的屏幕的设计。我的适配方案也是基于这个尺寸。

观察各个型号的设备，可以看到，不考虑iPhone X的情况下，4.7英寸的长宽比是最大的，因此按照667这个高度scale transform时，所有设备都能够完整显示。

![image](https://github.com/linzhiman/linzhiman.github.io/blob/master/resource/1808/iOS全屏幕适配方案-基于scale-transform-1.png?raw=true)

对于3.5英寸屏幕，缩放因子scale=480/667.0，宽度width=320/scale=445，即将view大小设置为445*667，scale后刚好充满整个屏幕；
对于4英寸屏幕，缩放因子scale=568/667.0，宽度width=320/scale=375，即将view大小设置为375*667，scale后刚好充满整个屏幕；
对于4.7英寸屏幕，不需要scale transform，将view大小设置为375*667即可；
对于5.5英寸屏幕，缩放因子scale=736/667.0，宽度width=414/scale=375，即将view大小设置为375*667，scale后刚好充满整个屏幕；
对于5.8英寸屏幕，不需要scale transform，将view大小设置为375*812即可；

以下为存在导航栏的场景下：

对于3.5英寸屏幕，缩放因子scale=(480-64)/(667.0-64)，宽度width=320/scale=464，即将view大小设置为464*603，scale后刚好充满整个屏幕；
对于4英寸屏幕，缩放因子scale=(568-64)/(667.0-64)，宽度width=320/scale=383，即将view大小设置为383*603，scale后刚好充满整个屏幕；
对于4.7英寸屏幕，不需要scale transform，将view大小设置为375*(667-64)即可；
对于5.5英寸屏幕，缩放因子scale=(736-64)/(667.0-64)，宽度width=414/scale=371，即将view大小设置为371*603，scale后刚好充满整个屏幕；
对于5.8英寸屏幕，不需要scale transform，将view大小设置为375*(812-64)即可；

总结以上场景，除了5.5英寸存在导航栏的场景，其他场景下以4.7英寸屏幕高度为基准，view宽度都超过375，可以完整显示view；
对于5.5英寸存在导航栏的场景，若内容可拉伸则可以完整显示，若不可拉伸且大于371则会导致超出屏幕；
有一点需要特别注意，view宽度不再是屏幕宽度，也不固定为375，而是不同型号设备有不同值。

具体实现可以参考一下代码：

![image](https://github.com/linzhiman/linzhiman.github.io/blob/master/resource/1808/iOS全屏幕适配方案-基于scale-transform-2.png?raw=true)

![image](https://github.com/linzhiman/linzhiman.github.io/blob/master/resource/1808/iOS全屏幕适配方案-基于scale-transform-3.png?raw=true)

demo：https://github.com/linzhiman/ATBaseViewController
