---
layout: post
title: UIViewController自动容器
---
# {{ page.title }}

App需求多变，常常过一段时间产品+设计就来个大改版，原先设计好的UI结构很难适应。
举个例子，一个设置页面，原先用UITableViewController实现的一个静态列表，产品希望增加一个头部，固定在列表上方。
如果列表不需要支持滚动，可以用HeaderView立马实现，然而列表就是足够长而需要滚动，HeaderView在列表下拉时会有弹簧动画，无法做到固定。
怎么解决这个问题呢？
第一个，重载scrollViewDidScroll。
将头部添加到UITableViewController.view，在滚动事件中修正头部位置，同时设置一个同样高度的占位HeaderView。头部添加手势的话可能与ScrollView冲突。
- (void)scrollViewDidScroll:(UIScrollView *)scrollView {
    CGRect frame = self.headView.frame;
    frame.origin.y = scrollView.contentOffset.y;
    self.headView.frame = frame;
    [self.view bringSubviewToFront:self.headView];
}
第二个，如果是Push出来的页面，可以把头部添加到self.navigationController.view，这样需要维护头部的显示隐藏逻辑。导航栏交互可能出现问题，例如右滑Pop页面可能导致导航栏UI错位。
第三个，重构UITableViewController为UIViewController+UITableView，头部加在UIViewController.view上。改动稍大。
第四个，将UITableViewController作为ChildController。改动较小。
这里介绍一个自动添加ParentController的方法—通用容器类ATContainerViewController，附代码实现。
图a
图b
使用：
1、在Storyboard中创建一个新的UIViewController，指定ClassName为ATContainerViewController，并设置Restoration Id为ATContainer+目标controller类名，如ATContainer+MyViewController。
2、ATContainerViewController的viewDidLoad中解析Restoration Id，获取目标类名MyViewController。
3、MyViewController实现ATContainerViewControllerProtocol协议，ATContainerViewController调用[MyViewController at_createInstance]创建MyViewController实例。
4、ATContainerViewController将MyViewController实例addChildViewController/addSubView。
5、完成。

