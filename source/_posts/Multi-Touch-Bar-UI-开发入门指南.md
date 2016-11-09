title: Multi-Touch Bar UI 开发入门指南
date: 2016-11-02 19:03:05
tags: [Cocoa]
---

### 大致

当你学会如何在你的 NSViewController 中用代码去初始化 Multi-Touch Bar 之后，我们就需要开始了解如何深入的开发 Touch Bar 上的 UI 了。根据 Apple 内部的一个说法就是 Multi-Touch Bar 实际上是运行了一个 watchOS 来管理它的视图和逻辑，所以与此同时我们也可以看到苹果官方为 Multi-Touch Bar 提供的一套 UI 控件，总共包含下面这些控件，第一个就是 Touch Bar。

![](https://ooo.0o0.ooo/2016/11/01/58181fbf140db.jpg)

- Touch Bar Fixed Space
- Touch Bar Flexible Space

这两个是用于在 UI 上用来填充的空间，Touch Bar 默认的 UI 是从左到右，默认左右就是挨着的，如果你需要将两个按钮什么的分开一些，定义他们之间的间隔，你需要在他们中间塞 Space。

- Touch Bar Popover


- Touch Bar Group

/// BarItem 的集合，可以一起受到约束和用户的 Customization。

- Touch Bar Other Items Proxy

/// 后面详细描述，这是和 Responder Chain 相关的一个控件，可以让响应链下层的 Touch Bar 和上层的 Touch Bar 一起显示。

- Touch Bar Scrubber

![https://ooo.0o0.ooo/2016/11/01/581837cc4601b.png](https://ooo.0o0.ooo/2016/11/01/581837cc4601b.png)

/// 这是一个放进 NSCustomTouchBarItem 的 NSScrubber 对象。

- Touch Bar Color Picker

![](https://ooo.0o0.ooo/2016/11/01/5818386c45463.png)

/// 一个 NSColorPickerTouchBarItem 对象。

- Touch Bar Character Picker

![](https://ooo.0o0.ooo/2016/11/01/581838cf69274.png)

/// 一个  NSCandidateListTouchBarItem 对象。

- Touch Bar Sharing Service Picker

![](https://ooo.0o0.ooo/2016/11/01/5818393054d89.png)

/// 一个 NSSharingServicePickerTouchBarItem 对象。

这里的 Scrubber，Color Picker 和 Character Picker 这些控件就会有些陌生了，虽然看过苹果发布会的同学们对他们应该也陌生不到哪里去。这几个控件在当时发布会都有展示。分别是滑动图片或文字选择器，颜色选择器，Emoji 表情选择器以及分享选择器，他们都有自己相应的 Delegate 来处理事件。

- Touch Bar Label
- Touch Bar Button
- Touch Bar Slider
- Touch Bar Segmented Control
- Touch Bar View

最后的五项对于大家都比较好理解，都是一些 AppKit 中的控件在 Touch Bar 中的移植，对于事件的响应和 AppKit 做法相同。

> 当你的 macOS 应用中包含了一些自带 Touch Bar 的控件的时候，例如 NSTextField，如果你没有自定义你的 Touch Bar，那么在 Touch Bar 上也会显示相应的控件。这是预设好的。

### NSTouchBar

你可以将 NSTouchBar 看成是一个 NSTouchBarItem 的数组。在运行的时候，对于不同的 NSResponder 对应的 Touch Bar 和这些 Touch Bar 所对应的 BarItem，都是用 Identifier 来标识的，尽量使用 reverse-DNS 方式的 Identifier，例如 `com.company-name.app-name.alphanumeric-ID`。

> 这里插一句，为了接下来的说明：由于现在并没有见到过新的 MBP 真机，尚不清楚 Touch Bar 提供给用户的 Customization 是什么样的机制，不过大概也就和 Finder 的自定义类似？按照官方文档来说，应该是用户在运行的时候可以选择显示或删除哪些 BarItem 或者修改 BarItem 的排列顺序，这大概也就是为什么 Touch Bar 的 UI 是始终向左靠齐的吧。

NSTouchBar 自身有一个 Identifier： **customizationIdentifier**，这是在运行时允许用户自定义 UI 的标识，如果不设置，用户就无法通过系统来自定义 Touch Bar 的 UI。

**customizationAllowedItemIdentifiers**，一个数组，在这里面存放的 Identifier 对应的 BarItem 是允许在运行时被用户修改样式的。

**customizationRequiredItemIdentifiers**，这个数组里面存放的是不能被移除，一定会加载的 BarItem 对应的 Identifier。

**defaultItemIdentifiers**，这是 Touch Bar 默认显示的全部 BarItem，这个参数是**必须**被初始化的。在这里面声明的 Identifier 随后就会在 `-itemForIdentifier:` 这个 Delegate 方法中初始化成 BarItem。给予 BarItem 的 Identifier 必须保持**全局唯一**，系统有另外四个预留的 Identifier：

- NSTouchBarItemIdentifierFixedSpaceSmall
- NSTouchBarItemIdentifierFixedSpaceLarge
- NSTouchBarItemIdentifierFlexibleSpace
- NSTouchBarItemIdentifierOtherItemsProxy

另外，NSTouchBar 还有一个只读的属性 **itemIdentifiers**，这个数组里包含的是运行时当前显示的 BarItem 对应的 Identifier，无论用户在运行的时候对 Touch Bar 做了何种的 Customization，你都可以通过这个属性获得当前的状态。

#### 如何让一个 BarItem 在 Touch Bar 上居中显示？

**principalItemIdentifier** 这个属性所对应的 BarItem 会在 Touch Bar **居中**，是所谓的最高优先级的 Item，如果有多层 Touch Bar 存在，也只会有最上层的 principalItem 显示出来。这里，官方文档中特别提到：

> Do not hard-code spacing in an attempt to ensure an item is centered. If you want a group of items to appear centered in the Touch Bar, designate the group NSTouchBarItem as the principal item.
>
> 不要尝试通过添加 Space 的方法来让一个 Item 居中，如果你需要让一组 Item 居中，你可以用一个 BarItem Group 将他们包起来然后将整个 Group 设置为居中。

#### NSTouchBarDelegate

**delegate**，NSTouchBarDelegate 只有一个方法，那就是  

```
- (nullable NSTouchBarItem *)touchBar:(NSTouchBar *)touchBar makeItemForIdentifier:(NSTouchBarItemIdentifier)identifier;
```

这个方法就是初始化所有 BarItem 的地方。一般来说写法就是根据 Identifier 来初始化对应的 BarItem，像这样

```
if ([identifier isEqualToString:@"..."])
{ 
	...
	return BarItem;
}
```

### Layout & Nesting

由于用户可以自定义需要显示的 Touch Bar 宽度，你定义好的 Touch Bar 宽度是可变的。所以苹果的建议是**在设计的时候不要按照确定的宽度来设计，而植入一些动态布局的方**案。如果你需要更宽的宽度，你可以使用 Popover，scrubber 或者 scroll view。

在 Layout 的时候，响应链上层的 Touch Bar 是可以包含下层的 Touch Bar 的，上面在综述的时候提到过的 **Other Items Proxy** 就是用来实现这个功能的。只要你在初始化 Touch Bar 的时候为它的 defaultIdentifiers 加上 NSTouchBarItemIdentifierOtherItemsProxy，他就会在运行时**需要的时候在这个 Identifier 所在的位置**包含上来自下层的合适的 Touch Bar。这个**需要的时候**指的是下层的 NSResponder 调用 becomeFirstResponder 的时候。例如有一个 ViewController 自身带有一个 Touch Bar，而这个 ViewController 中带有一个 NSTextView，那么当点击这个 NSTextview ，也就是**响应链下层 NSResponder 调用 becomeFirstResponder** 的时候，Touch Bar 显示的是：

![](https://ooo.0o0.ooo/2016/11/01/5818435fa4fdd.png)

而如果焦点从 TexView 失去，则恢复成 ViewController 这一层的 Touch Bar：

![](https://ooo.0o0.ooo/2016/11/01/58184323610b8.png)

如果一个 Touch Bar 没有植入 NSTouchBarItemIdentifierOtherItemsProxy ，那么如果在响应链更底层出现了一个可以加载的 Touch Bar，那么上层的 Touch Bar 都会被隐藏。哦，并且 NSTouchBar 的 visible 属性是可以 KVO 的。

#### Responder Chain Search

所以 Touch Bar UI 的加载顺序和 UIView 的响应链类似，都是穿过整个响应链，从最底层开始向上寻找，如果遇到了 otherItemsProxy ， 则会将两者合并起来之后继续往上遍历，直到找到最上层，生成一个最后加载出来的 Touch Bar。

> Touch bars 的加载是通过查找指定的遵循了 NSTouchBarProvider Protocol 的组件来完成的。按照从响应链上层往下的顺序依次是：
>
> - the application delegate
> - the application object itself
> - the main window’s window controller
> - the main window’s delegate
> - the main window itself
> - the main window’s first responder
> - the key window’s window controller
> - the key window’s delegate
> - the key window itself
> - the key window’s first responder
>
> 如果上面这些对象是 NSResponder 或者 NSResponder 的子类，以该对象为起点的响应链也会被包括进来。例如 AppDelegate 原本是 NSObject，但是[在 iOS 上 AppDelegate 就是继承于 UIResponder](http://stackoverflow.com/questions/6893221/why-does-appdelegate-inherit-from-uiresponder) ，这是因为这样 AppDelegate 就可以成为整个响应链的最上层了。因此**如果我们将 AppDelegate 改为继承于 NSResponder** ，那么在 AppDelegate 层也可以加上一层 Touch Bar。 
>
> 例如在一个一般结构的应用中，这个响应链的查找顺序是这样的：
>
> - Application delegate
> - Application
> - key window controller
> - key window delegate
> - key window
> - view controller (closest to root of window)
> - view (closest to root of window)
> - intermediate view controllers and views
> - key window’s first responder’s view controller
> - key window’s first responder

#### Customization Boundary

那么对于最后显示的 Touch Bar 来说，每个 Touch Bar 的 Customization 属性依然管用。如果是 A Bar包含了 B Bar，那么对于 B Bar 的 Customization 是不能超出 B Bar 的范围的。例如有：

```
bar.defaultItemIdentifiers =@[IdentifierA,IdentifierB, NSTouchBarItemIdentifierOtherItemsProxy,IdentifierC];
```

这个 Bar 假设为 A Bar，在他的下层有一个 B Bar，那么在 AB 同时显示的时候，如果对 B Bar 的 Item 进行 Customization，那么是不能移动 IdentifierA，IdentifierB 和 IdentifierC 的位置的。

#### Visual Priority

**视图优先级**。由于你所开发的 Touch Bar 在运行的时候有可能被用户修改宽度，因此如果你有一排 BarItem，他们会从右往左依次被挤掉，这时候如果右边有一些重要的 BarItem 存在，你就需要为不同的 BarItem 设置不同的视图优先级，这样即使宽度变化，在重新 Layout 的时候也会按照定义好的视图优先级排列 BarItem。

系统提供了三种默认的枚举类型视图优先级，分别为 NSTouchBarItemPriorityHigh， NSTouchBarItemPriorityNormal 和 NSTouchBarItemPriorityLow，对应了 1000，0 和 -1000 的 float 类型的值，因此你也可以直接为 BarItem 设置 float 类型的优先级。

### 用 Storyboard 来定义你的 Multi-Touch Bar

讲了这么多，用代码生成 Touch Bar 多累啊，我们来看看如何用 Storyboard 来创建吧。

![](https://ooo.0o0.ooo/2016/11/01/581844c5bc04f.jpg)

方法很简单，直接往你的 ViewController 里面从右下角拖一个 Touch Bar 过来就行了。然后往里面添加控件，对于不同的控件，Delegate，Datasource 和控件的触发事件，都是用 Control-Drag 的方法将引用加到你的 .m 文件里去。

### 各种控件的具体用法

接下来该是一个个来讲控件了，也是大家在之后的开发中最实用的部分啦。

#### NSColorPickerTouchBarItem

colorPicker 有三种初始化方法：

- colorPickerWithIdentifier
- textColorPickerWithIdentifier
- strokeColorPickerWithIdentifier

这三种方法的区别其实就是在按钮显示的图标上，一个是常规的颜色选择器，第二个代表的是字体颜色选择器，第三种初始化代表的是边框颜色选择器。初始化之后的逻辑都是一样的：

```
colorPickerItem.target = self;
colorPickerItem.action = @selector(ColorAction:);
```

然后在 `ColorAction` 这个方法中获得选择到的颜色：

```
((NSColorPickerTouchBarItem *)sender).color
```

**@property colorList**

你也可以通过修改 **colorList** 这个属性来自定义 Color Picker 能够选取的颜色列表。如果不设置这个属性，则默认显示的是系统的取色盘，各种颜色都有：

![](https://ooo.0o0.ooo/2016/11/02/5819676729847.png)

当给了一个 list 之后：

```
colorPickerItem.colorList = [[NSColorList alloc] init];
[colorPickerItem.colorList setColor:[NSColor redColor] forKey:@"Red"];
[colorPickerItem.colorList setColor:[NSColor greenColor] forKey:@"Green"];
[colorPickerItem.colorList setColor:[NSColor blueColor] forKey:@"Blue"];
```

显示的就是这样：

![](https://ooo.0o0.ooo/2016/11/02/58196802031d6.png)

#### NSSliderTouchBarItem

初始化：

```
sliderTouchBarItem.slider.minValue = 0.0f;
sliderTouchBarItem.slider.maxValue = 100.0f;
sliderTouchBarItem.slider.continuous = YES;
sliderTouchBarItem.target = self;
sliderTouchBarItem.action = @selector(sliderChanged:);
sliderTouchBarItem.label = @"Slider";
sliderTouchBarItem.customizationLabel = @"Slider";
```

这里面的 action 会在 slider 的值每一次发生改变的时候调用。

值得一提的是 SliderItem 还提供了两个 AccessoryView 的自定义空间，左右各一个，分别叫做 **minimumValueAccessory** 和 **maximumValueAccessory**，可以用 NSImage 来初始化他们：

```
NSSliderAccessory *minSliderAccessory = [NSSliderAccessory accessoryWithImage:image];
```

其他的自定义都是围绕着 NSSlider 做的了。

#### NSScrubber

这是一个为了 Touch Bar 而全新创建的控件，在发布会上有着重的展示。他需要单独的添加到一个 TouchBarItem 里去：

```
scrubber = [[NSScrubber alloc] initWithFrame:NSMakeRect(0, 0, 310, 30)];
// Height should be 30 
scrubber.delegate = self;   
scrubber.dataSource = self; 
```

NSScrubber 类似 UICollectionView ，也是有基于 ItemView 的复用机制的，你可以继承 NSScrubberItemView 来写你自己需要的 CustomItemView。然后在初始化之后需要为 ItemView 加上一个复用的标识：

```
[scrubber registerClass:[NSScrubberItemView class] forItemIdentifier:ScrubberItemIdentifier];
```

**@property showsAdditionalContentIndicators** 

代表的是当左右滑动还有更多内容的时候，在左右侧会显示一块渐变的蒙层，用来表示滑动还有更多内容，你可以选择是否显示。

**@property selectedIndex** 

即当前选中的 ItemView 的 index，可以直接设置。

**@property showsArrowButtons** 

代表的是是否显示左右移动的箭头按钮，如果显示的话你也可以点击左右箭头的按钮来滑动列表。

**@property backgroundColor** 

NSScrubber 的背景颜色，显示在内容之后，如果 **backgroundView** 为空才会显示，同时也是只有 NSScrubberTextItemView 类型的 ItemView 才会显示出后面的背景颜色。

**@property DataSource**

```
- (NSInteger)numberOfItemsForScrubber:(NSScrubber *)scrubber
{
	return 1;
}

- (NSScrubberItemView *)scrubber:(NSScrubber *)scrubber viewForItemAtIndex:(NSInteger)index
{
	NSScrubberItemVie *itemView = [scrubber makeItemWithIdentifier:ScrubberItemIdentifier owner:nil];
	return itemView;
}
```

数据源加载比较简单，一个给 Item 的总数，另一个给 ItemView 就行，上面给出了最简单的示例。

**@property Delegate**

NSScrubber 有好几个 Delegate 可以调用，也都比较好理解，一般来说实现第一个选中回调就足够了：

- didSelectItemAtIndex:(NSInteger)selectedIndex 
- didHighlightItemAtIndex:(NSInteger)highlightedIndex 
- didChangeVisibleRange:(NSRange)visibleRange
- didBeginInteractingWithScrubber:(NSScrubber *)scrubber
- didFinishInteractingWithScrubber:(NSScrubber *)scrubber
- didCancelInteractingWithScrubber:(NSScrubber *)scrubber

**@property mode**

这个代表的是 NSScrubber 的滑动模式，一共就两种，**NSScrubberModeFree** 和 **NSScrubberModeFixed**。

Free 模式下的 Scrubber 可以自由的滑动，但是你必须在**滑动之后再次单击 Item** 来选中，而 Fixed 模式下的 Scrubber 是不能自由滑动的，但是对于 ItemView 的选中是**跟随手指的拖动**的，不需要让手指离开 Touch Bar 再次点击选中。对于宽度固定，或者不需要用户再滑出范围的 Scrubber 可以使用 Fixed 模式。

**@property SelectionStyle**

Scrubber 给选中样式提供了很方便的自定义属性，有 **selectionBackgroundStyle** 和 **selectionOverlayStyle**，同时也可以通过设置 **floatsSelectionViews** 为 YES 让选中 View 切换的时候可以加上从 A 点到 B 点流畅的移动动画，而不是从 A 点消失然后从 B 点出现这样的动画。

selectionBackgroundStyle 和 selectionOverlayStyle 分别是选中的时候**背景**和**表面蒙层**的样式，只提供了 outlineOverlayStyle

![](https://ooo.0o0.ooo/2016/11/02/5819b0f9959ad.png)

和 roundedBackgroundStyle 两种默认样式：

![](https://ooo.0o0.ooo/2016/11/02/5819b17f754fb.png)

当然，这个 SelectionStyle 也是高度自定义的，但是你需要继承两个视图的基类来实现：

1. 定义一个继承于 NSScrubberSelectionStyle 的 CustomStyle 
2. 重载 `-(nullable __kindof NSScrubberSelectionView *)makeSelectionView` 这个方法
3. 定义一个继承于 NSScrubberSelectionView 的 CustomSelectionView
4. 自定义 CustomSelectionView 的样式
5. 在重载的方法中返回 CustomSelectionView

**@property scrubberLayout**

类似于 UICollectionView ，NSScrubber 的 layout 也是由 **NSScrubberLayout** 来控制的。系统提供了两种默认的 Layout，**NSScrubberFlowLayout** 和 **NSScrubberProportionalLayout**。前者可以通过 LayoutDelegate 来给出每一个 Item 所占的宽度，后者则不需要通过 delegate 而是直接将 Item 的宽度设置为 Scrubber 的可视区域按照 Item 的数量平分之后的宽度。你可以这样设置：

```
NSScrubberLayout *scrubberLayout;
scrubberLayout = [[NSScrubberFlowLayout alloc] init];
// or:
scrubberLayout = [[NSScrubberProportionalLayout alloc] init];
scrubber.scrubberLayout = scrubberLayout;
```

对于 NSScrubberFlowLayoutDelegate 你需要实现这一个方法：

```
- (NSSize)scrubber:(NSScrubber *)scrubber layout:(NSScrubberFlowLayout *)layout sizeForItemAtIndex:(NSInteger)itemIndex
{
    return NSMakeSize(width, 30);
}
```

#### NSSharingServicePickerTouchBarItem

NSSharingServicePickerTouchBarItem 的 delegate 只有一个委托方法需要实现：

```
- (NSArray *)itemsForSharingServicePickerTouchBarItem:(NSSharingServicePickerTouchBarItem *)pickerTouchBarItem
{
    return @[...];
}
```

在这里返回的对象一定要是能够支持 NSPasteboardWriting 协议或者是继承于 NSItemProvider 的对象，例如文本，图片或者 NSURL，可以参考 [NSItemProvider 的官方文档](https://developer.apple.com/reference/foundation/nsitemprovider?language=objc)。然后就会在分享的时候传到分享的应用里去。

#### NSPopoverTouchBarItem

首先，我们来看如何自定义 Popover 在未点击状态下的显示 Button。

- collapsedRepresentation 如果不为空，则显示一个自定义的 NSView，如果为空，则加载不为空的下面两个属性
- collapsedRepresentationImage 给 Popover 的 button 添加一个 NSImage
- collapsedRepresentationLabel 给 Popover 的 button 添加一个 **NSString**

Popover 其实是在点击按钮之后弹出一个**新的 Touch Bar**，你需要再初始化一个新的 Touch Bar 并在同样的 Delegate 中返回对应的 BarItem，注意它和你之前的 Touch Bar 不是响应链上下层的关系，因此不会出现 Nesting 即上层的 Touch Bar 包含下层的 Touch Bar，而是完全覆盖了下面的 Touch Bar。如果你将新的 Touch Bar 赋值给了 **pressAndHoldTouchBar** 而不是 **popoverTouchBar**，你还可以实现按住按钮显示 Popover 且松开手指返回。

#### Touch Bar Image Template

最后还要为大家提供一个便利，苹果为了支持 Touch Bar 的开发，同时也为了统一设计样式，在 NSImage 中添加了非常多 Button 的模板图片，由于提供的模板太多了，不一一列出，就把列表附上，如果有同学开发的时候有需要可以去 NSImage 的头文件里面找，如果系统模板有的话可以直接使用。

![](https://ooo.0o0.ooo/2016/11/02/5819c14c90072.png)

### 最后

Touch Bar 开发的新特性和坑肯定还有很多，现在新的 MBP 都还没发货，相信等更多的开发者开始真正开发上面的插件之后会有更丰富的经验和总结产生，我这篇博客也就是投石问路。然而，对 Touch Bar 十分感兴趣的我已经等不及开始研究它并且开始准备在上面做一些有意思的东西了！