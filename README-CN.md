像SwiftUI一样使用StackUI。


# 特性

- [x] 类似SwiftUI的声明式语法;
- [x] 数据驱动UI，更新数据UI自动更新；
- [x] 支持HScrollStack，VScrollStack，当内容超过Stack的宽度或高度时，自动开启滚动；
- [x] 链式语法配置UIKit;


# 要求

- iOS 9.0+
- XCode 13.0+
- Swift 5.0+

# 安装


## CocoaPods

```ruby
target '<Your Target Name>' do
    pod 'StackUI'
end
```
先执行`pod repo update`，再执行`pod install`

## SPM
在项目的`Package.swift`文件添加`dependencies`
```
dependencies: [
    .package(url: "https://github.com/pujiaxin33/StackUI.git", .upToNextMajor(from: "0.0.1"))
]
```

# 使用示例

## 扩展`ThemeStyle`添加自定义style
`ThemeStyle`内部仅提供了一个默认的`unspecified`style，其他的业务style需要自己添加，比如只支持`light`和`dark`，代码如下：
```Swift
extension ThemeStyle {
    static let light = ThemeStyle(rawValue: "light")
    static let dark = ThemeStyle(rawValue: "dark")
}
```

## 基础使用
```Swift
view.theme.backgroundColor = ThemeProvider({ (style) in
    if style == .dark {
        return .white
    }else {
        return .black
    }
})
imageView.theme.image = ThemeProvider({ (style) in
    if style == .dark {
        return UIImage(named: "catWhite")!
    }else {
        return UIImage(named: "catBlack")!
    }
})
```

## 自定义属性配置
如果库没有原生支持某个属性，可以在customization里面统一处理。
```Swift
view.theme.customization = ThemeProvider({[weak self] style in
    //可以选择任一其他属性
    if style == .dark {
        self?.view.bounds = CGRect(x: 0, y: 0, width: 30, height: 30)
    }else {
        self?.view.bounds = CGRect(x: 0, y: 0, width: 80, height: 80)
    }
})
```

## extension ThemeWrapper添加属性

如果某一个属性在项目中经常使用，使用上面的**自定义属性配置**觉得麻烦，就可以自己extension ThemeWrapper添加想要的属性。（ps：你也可以提交一个Pull Request申请添加哟）

下面是UILabel添加shadowColor的示例：
```Swift
//自定义添加ThemeProperty，目前仅支持UIView、CALayer、UIBarItem及其它们的子类
extension ThemeWrapper where Base: UILabel {
    var shadowColor: ThemeProvider<UIColor>? {
        set(new) {
            let baseItem = self.base
            ThemeTool.setupViewThemeProperty(view: self.base, key: "UILabel.shadowColor", provider: new) {[weak baseItem] (style) in
                baseItem?.shadowColor = new?.provider(style)
            }
        }
        get { return ThemeTool.getThemeProvider(target: self.base, with: "UILabel.shadowColor") as? ThemeProvider<UIColor> }
    }
}
```
调用还是一样的：
```Swift
//自定义属性shadowColor
shadowColorLabel.shadowOffset = CGSize(width: 0, height: 2)
shadowColorLabel.theme.shadowColor = ThemeProvider({ style in
    if style == .dark {
        return .red
    }else {
        return .green
    }
})
```

## 配置封装示例
`JXTheme`是一个提供主题属性配置的轻量级基础库，不限制使用哪种方式加载资源。下面提供的三个示例仅供参考。

### ThemeProvider自定义初始化器
比如在项目中添加如下代码：
```Swift
extension ThemeProvider {
    //根据项目支持的ThemeStyle调整
    init(light: T, dark: T) {
        self.init { style in
            switch style {
            case .light: return light
            case .dark: return dark
            default: return light
            }
        }
    }
}
```
在业务代码中调用：
```Swift
tableView.theme.backgroundColor = ThemeProvider(light: UIColor.white, dark: UIColor.white)
```
这样就可以避免ThemeProvider闭包的形式，更加简洁。

### 根据枚举定义封装示例

一般的换肤需求，都会有一个UI标准。比如`UILabel.textColor`定义三个等级，代码如下：
```Swift
enum TextColorLevel: String {
    case normal
    case mainTitle
    case subTitle
}
```
然后可以封装一个全局函数传入`TextColorLevel`返回对应的配置闭包，就可以极大的减少配置时的代码量，全局函数如下：
```Swift
func dynamicTextColor(_ level: TextColorLevel) -> ThemeProvider<UIColor> {
    switch level {
    case .normal:
        return ThemeProvider({ (style) in
            if style == .dark {
                return UIColor.white
            }else {
                return UIColor.gray
            }
        })
    case .mainTitle:
        ...
    case .subTitle:
        ...
    }
}
```
主题属性配置时的代码如下：
```Swift
themeLabel.theme.textColor = dynamicTextColor(.mainTitle)
```

### 本地Plist文件配置示例
与**常规配置封装**一样，只是该方法是从本地Plist文件加载配置的具体值，具体代码参加`Example`的`StaticSourceManager`类

### 根据服务器动态添加主题
与**常规配置封装**一样，只是该方法是从服务器加载配置的具体值，具体代码参加`Example`的`DynamicSourceManager`类

## 有状态的控件
某些业务需求会存在一个控件有多种状态，比如选中与未选中。不同的状态对于不同的主题又会有不同的配置。配置代码参考如下：
```Swift
statusLabel.theme.textColor = ThemeProvider({[weak self] (style) in
    if self?.statusLabelStatus == .isSelected {
        //选中状态一种配置
        if style == .dark {
            return .red
        }else {
            return .green
        }
    }else {
        //未选中状态另一种配置
        if style == .dark {
            return .white
        }else {
            return .black
        }
    }
})
```

当控件的状态更新时，需要刷新当前的主题属性配置，代码如下：
```Swift
func statusDidChange() {
    statusLabel.theme.textColor?.refresh()
}
```

如果你的控件支持多个状态属性，比如有`textColor`、`backgroundColor`、`font`等等，你可以不用一个一个的主题属性调用`refresh`方法，可以使用下面的代码完成所有配置的主题属性刷新：
```Swift
func statusDidChange() {
    statusLabel.theme.refresh()
}
```

## overrideThemeStyle
不管主题如何切换，`overrideThemeStyleParentView`及其子视图的`themeStyle`都是`dark`
```Swift 
overrideThemeStyleParentView.theme.overrideThemeStyle = .dark
```

# 实现原理

- [实现原理](https://github.com/pujiaxin33/JXTheme/blob/master/Document/%E5%8E%9F%E7%90%86.md)


# 其他说明

## 为什么使用`theme`命名空间属性，而不是使用`theme_xx`扩展属性呢？
- 如果你给系统的类扩展了N个函数，当你在使用该类时，进行函数索引时，就会有N个扩展的方法干扰你的选择。尤其是你在进行其他业务开发，而不是想配置主题属性时。
- 像`Kingfisher`、`SnapKit`等知名三方库，都使用了命名空间属性实现对系统类的扩展，这是一个更`Swift`的写法，值得学习。

## 主题切换通知
```Swift
extension Notification.Name {
    public static let JXThemeDidChange = Notification.Name("com.jiaxin.theme.themeDidChangeNotification")
}
```

## `ThemeManager`根据用户ID存储主题配置

```
/// 配置存储的标志key。可以设置为用户的ID，这样在同一个手机，可以分别记录不同用户的配置。需要优先设置该属性再设置其他值。
public var storeConfigsIdentifierKey: String = "default"
```

## 迁移到系统API指南
当你的应用最低支持iOS13时，如果需要的话可以按照如下指南，迁移到系统方案。
[迁移到系统API指南，点击阅读](https://github.com/pujiaxin33/JXTheme/blob/master/Document/%E8%BF%81%E7%A7%BB%E5%88%B0%E7%B3%BB%E7%BB%9FAPI%E6%8C%87%E5%8D%97.md)

# 目前支持的类及其属性

这里的属性是有继承关系的，比如`UIView`支持`backgroundColor`属性，那么它的子类`UILabel`等也就支持`backgroundColor`。如果没有你想要支持的类或属性，欢迎提PullRequest进行扩展。

## UIView

- `backgroundColor`
- `tintColor`
- `alpha`
- `customization`

## UILabel

- `font`
- `textColor`
- `shadowColor`
- `highlightedTextColor`
- `attributedText`

## UIButton

- `func setTitleColor(_ colorProvider: ThemeColorDynamicProvider?, for state: UIControl.State)`
- `func setTitleShadowColor(_ colorProvider: ThemeColorDynamicProvider?, for state: UIControl.State)`
- `func setAttributedTitle(_ textProvider: ThemeAttributedTextDynamicProvider?, for state: UIControl.State)`
- `func setImage(_ imageProvider: ThemeImageDynamicProvider?, for state: UIControl.State)`
- `func setBackgroundImage(_ imageProvider: ThemeImageDynamicProvider?, for state: UIControl.State)`

## UITextField

- `font`
- `textColor`
- `attributedText`
- `attributedPlaceholder`
- `keyboardAppearance`

## UITextView

- `font`
- `textColor`
- `attributedText`
- `keyboardAppearance`

## UIImageView

- `image`

## CALayer

- `backgroundColor`
- `borderColor`
- `borderWidth`
- `shadowColor`
- `customization`

## CAShapeLayer

- `fillColor`
- `strokeColor`

## UINavigationBar

- `barStyle`
- `barTintColor`
- `titleTextAttributes`
- `largeTitleTextAttributes`

## UITabBar

- `barStyle`
- `barTintColor`
- `shadowImage`

## UISearchBar

- `barStyle`
- `barTintColor`
- `keyboardAppearance`

## UIToolbar

- `barStyle`
- `barTintColor`

## UISwitch

- `onTintColor`
- `thumbTintColor`

## UISlider

- `thumbTintColor`
- `minimumTrackTintColor`
- `maximumTrackTintColor`
- `minimumValueImage`
- `maximumValueImage`

## UIRefreshControl

- `attributedTitle`

## UIProgressView

- `progressTintColor`
- `trackTintColor`
- `progressImage`
- `trackImage`

## UIPageControl

- `pageIndicatorTintColor`
- `currentPageIndicatorTintColor`

## UIBarItem

- `func setTitleTextAttributes(_ attributesProvider: ThemeAttributesDynamicProvider?, for state: UIControl.State)`
- `image`

## UIBarButtonItem

- `tintColor`

## UIActivityIndicatorView

- `style`
- `color`

## UIScrollView

- `indicatorStyle`

## UITableView

- `separatorColor`
- `sectionIndexColor`
- `sectionIndexBackgroundColor`

# Contribution

有任何疑问或建议，欢迎提Issue和Pull Request进行交流🤝



