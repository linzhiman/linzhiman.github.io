---
layout: post
title: UIViewController自动容器
---
# {{ page.title }}

App需求多变，常常过一段时间产品+设计就来个大改版，原先设计好的UI结构很难适应。

举个例子，一个设置页面，原先用UITableViewController实现的一个静态列表，产品希望增加一个头部，固定在列表上方。

如果列表不需要支持滚动，可以用HeaderView立马实现，然而列表就是足够长而需要滚动，HeaderView在列表下拉时会有弹簧动画，无法做到固定。

怎么解决这个问题呢？

第一个，重载scrollViewDidScroll。将头部添加到UITableViewController.view，在滚动事件中修正头部位置，同时设置一个同样高度的占位HeaderView。头部添加手势的话可能与ScrollView冲突。

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

    //
    //  ATContainerViewController.h
    //  AppTemplateLib
    //
    //  Created by linzhiman on 2017/6/19.
    //  Copyright © 2017年 zhiniu. All rights reserved.
    //
    
    #import <UIKit/UIKit.h>
    
    @protocol ATContainerViewControllerProtocol <NSObject>
    + (id)at_createInstance;
    @end

    /**
     Storyboard中指定UIViewController的ClassName为ATContainerViewController，并设置Restoration Id为ATContainer+目标controller类名，        目录类需实现ATContainerViewControllerProtocol，后续将调用at_createInstance方法创建目标实例，并addChildViewController/addSubView。
     */
    @interface ATContainerViewController : UIViewController
    
    @end


    //
    //  ATContainerViewController.m
    //  AppTemplateLib
    //
    //  Created by linzhiman on 2017/6/19.
    //  Copyright © 2017年 zhiniu. All rights reserved.
    //

    #import "ATContainerViewController.h"
    #import "ATGlobalMacro.h"

    ATConstStringDefine(ATContainer_Prefix, @"ATContainer+")

    @interface ATContainerViewController ()

    @property (nonatomic, strong) UIViewController<ATContainerViewControllerProtocol> *subViewController;

    @end

    @implementation ATContainerViewController

    - (void)viewDidLoad {
        [super viewDidLoad];
        // Do any additional setup after loading the view.
        
        NSString *restorationIdentifier = self.restorationIdentifier;
        if ([restorationIdentifier hasPrefix:ATContainer_Prefix]) {
            NSString *className = [restorationIdentifier substringFromIndex:ATContainer_Prefix.length];
            Class aClass = NSClassFromString(className);
            if ([aClass conformsToProtocol:@protocol(ATContainerViewControllerProtocol) ]) {
                id instance = [aClass at_createInstance];
                if ([instance isKindOfClass:[UIViewController class]]) {
                    _subViewController = instance;
                    [self addChildViewController:_subViewController];
                    _subViewController.view.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
                    [self.view addSubview:_subViewController.view];
                }
            }
        }
    }

    - (void)didReceiveMemoryWarning {
        [super didReceiveMemoryWarning];
        // Dispose of any resources that can be recreated.
    }

    /*
    #pragma mark - Navigation

    // In a storyboard-based application, you will often want to do a little preparation before navigation
    - (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender {
        // Get the new view controller using [segue destinationViewController].
        // Pass the selected object to the new view controller.
    }
    */

    @end


使用：
1、在Storyboard中创建一个新的UIViewController，指定ClassName为ATContainerViewController，并设置Restoration Id为ATContainer+目标controller类名，如ATContainer+MyViewController。

2、ATContainerViewController的viewDidLoad中解析Restoration Id，获取目标类名MyViewController。

3、MyViewController实现ATContainerViewControllerProtocol协议，ATContainerViewController调用[MyViewController at_createInstance]创建MyViewController实例。

4、ATContainerViewController将MyViewController实例addChildViewController/addSubView。

5、完成。

