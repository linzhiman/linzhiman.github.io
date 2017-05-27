1、在Github上建好自己的开源项目。

如我的项目AppTemplateLib。附地址 https://github.com/linzhiman/AppTemplateLib
创建项目需要加上一个LICENSE，或者后续再添加上LICENSE文件。

2、创建.podspec文件。

打开终端，cd到工程根目录下，执行 pod spec create NAME  

    pod spec create AppTemplateLib

以上命令会生成一个AppTemplateLib.podspec文件。该文件描述了项目及作者信息，源码/库/资源，以及依赖的系统库/第三方库等配置信息。
打开文件查看，可以发现里面有相当完备的说明，根据说明修改相关配置信息。

以下为我的.podspec文件，项目提供所有源码，且没有资源文件：  

    Pod::Spec.new do |s|  
      s.name         = "AppTemplateLib"  
      s.version      = "0.0.2"  
      s.summary      = "AppTemplateLib for quick start."  
      s.homepage     = "https://github.com/linzhiman/AppTemplateLib"  
      s.license      = "MIT"  
      s.author             = { "linzhiman" => "154298785@qq.com" }  
      s.platform     = :ios, "7.0"  
      s.source       = { :git => "https://github.com/linzhiman/AppTemplateLib.git", :tag => "#{s.version}" }  
      s.source_files  = "AppTemplateLib", "AppTemplateLib/**/*.{h,m}"  
    end

这里把所有文件都放在一个目录下面，如果你有分目录的话，可以采用下面的方式，不过需要保证subspec独立可以编译，用户可以只pod其中部分subspec。 

    Pod::Spec.new do |s|  
      s.name         = "AppTemplateLib" 
      s.version      = "0.0.5"  
      s.summary      = "AppTemplateLib for quick start."  
      s.homepage     = "https://github.com/linzhiman/AppTemplateLib"  
      s.license      = "MIT"  
      s.author             = { "linzhiman" => "154298785@qq.com" }  
      s.platform     = :ios, "7.0"  
      s.source       = { :git => "https://github.com/linzhiman/AppTemplateLib.git", :tag => "#{s.version}" }  
      s.default_subspecs = "Utility", "Model", "UI" 
      s.subspec 'Utility' do |sp| 
        sp.source_files = "AppTemplateLib/Utility/**/*.{h,m}" 
      end 
      s.subspec 'Model' do |sp| 
        sp.source_files = "AppTemplateLib/Model/**/*.{h,m}" 
        sp.dependency "AppTemplateLib/Utility"  
      end 
      s.subspec 'UI' do |sp|  
        sp.source_files = "AppTemplateLib/UI/**/*.{h,m}"  
        sp.dependency "AppTemplateLib/Utility"  
      end 
    end 

修改好配置信息后，需要执行pod命令验证一下，没问题的话将会看到 AppTemplateLib.podspec passed validation.  

    pod spec lint AppTemplateLib.podspec

![image](https://github.com/linzhiman/linzhiman.github.io/blob/master/resource/1705/CocoaPods发布Github开源项目-1.jpg?raw=true)

3、发布。

有了上面的.podspec文件，可以保存到本地CocoaPod目录中，默认为~/.cocoapods/repos/master/Specs，这样本地可以通过CocoaPods来引用这个库了。如果需要发布给其他人使用，需要提交到CocoaPods代码库中。

首先，注册trunk  

    pod trunk register 154298785@qq.com linzhiman

后面的邮件地址和用户名，修改为自己，执行完命令会收到一封验证邮件，需要打开验证一下。

![image](https://github.com/linzhiman/linzhiman.github.io/blob/master/resource/1705/CocoaPods发布Github开源项目-2.jpg?raw=true)

![image](https://github.com/linzhiman/linzhiman.github.io/blob/master/resource/1705/CocoaPods发布Github开源项目-3.jpg?raw=true)

接着，发布，在.podspec目录中执行以下命令： 

    pod trunk push

当你看到下面的执行结果说明发布成功了，通过下面命令也可以查询到了： 

    pod search AppTemplateLib

![image](https://github.com/linzhiman/linzhiman.github.io/blob/master/resource/1705/CocoaPods发布Github开源项目-4.jpg?raw=true)

至此，所有流程全部完成！大家都可以pod install了。

问答：
1、修改了代码，需要更新怎么处理？
答：在项目架构不变情况下，只需要在Github上用新版本打tag，然后修改.podspec中的s.version。如果新增了资源文件，依赖库则需要对应修改。然后重新push即可。s.version每次trunk push需要递增，重复的话将会失败。  

    [!] Unable to accept duplicate entry for: AppTemplateLib (0.0.4)

2、除了tag方式，还有其他配置方式吗？
答：看说明很详细，Supports git, hg, bzr, svn and HTTP.举两个例子：
[1]Git的commit方式 

    s.source = { :git => "https://github.com/linzhiman/AppTemplateLib.git", :commit => "69794d422501133342b6edc0890f05df000eec64" }

[2]Http方式 

    s.source = { :http => "http://abc.xyz.com/123.zip" }

3、项目提供了资源文件怎么处理？
答：
[1]直接提供素材或bundle，建议文件名加前缀 

    s.resource  = "icon.png"
    或 s.resources = "Resources/*.png"
    与 s.preserve_paths = "FilesToSave", "MoreFilesToSave"

[2]需要编译工程才输出的资源包
暂时发现只能编译后再用[1]方式配置。
注：建议素材采用bundle的方式，相当于加了名字空间，这样无需每个素材名字加前缀。

4、依赖系统库？  

    s.framework  = "SomeFramework"
    s.frameworks = "SomeFramework", "AnotherFramework"
    s.library   = "iconv"
    s.libraries = "iconv", "xml2"

5、依赖其他第三方库？ 

    s.dependency "JSONKit", "~> 1.4"
