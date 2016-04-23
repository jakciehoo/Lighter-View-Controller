******iOS架构师之路：控制器（View Controller）瘦身设计******

**前言**

　　古老的MVC架构是容易被iOS开发者理解和接受的设计模式，但是由于iOS开发的项目功能越来越负责庞大，项目代码也随之不断壮大，MVC的模糊定义导致我们的业务开发工程师很容易把大量的代码写到视图控制器中，行业中对这种控制器有个专业词汇Massive ViewControler（臃肿的视图控制器）。代码臃肿导致可读性可维护性差，而且这种不清晰的设计还有许多的副作用，比如代码重用性差。作为架构师需要关注项目的代码质量。指导业务开发工程师写出高质量，高健壮性，高可用的代码也是很重要的工作。因此需要知道一些为控制器瘦身的技巧，并在项目中帮助业务开发工程师合理的运用它们。本文翻译一篇国外优秀文章：[Lighter View Controllers](https://www.objc.io/issues/1-view-controllers/lighter-view-controllers/)。
　　
原文地址：[iOS架构师之路：控制器（View Controller）瘦身设计](http://www.jianshu.com/p/0630fd7f9a44)


** **

**分离数据源(Data Source)等协议(Protocol)**

　　瘦身控制器的有效方法之一就是将实现 UITableViewDataSource 协议相关的代码封装成一个类(比如本文中的 ArraryDataSource )。如果你多用几次这个设计，你就会创建复用性高的封装类。
　　举个例子，示例工程中的类 PhotosViewController实现如下数据源方法：
```python

# pragma mark Private Method

- (Photo*)photoAtIndexPath:(NSIndexPath*)indexPath {
    return photos[(NSUInteger)indexPath.row];
}

# pragma mark UITableViewDataSource

- (NSInteger)tableView:(UITableView*)tableView
 numberOfRowsInSection:(NSInteger)section {
    return photos.count;
}

- (UITableViewCell*)tableView:(UITableView*)tableView
        cellForRowAtIndexPath:(NSIndexPath*)indexPath {
    PhotoCell* cell = [tableView dequeueReusableCellWithIdentifier:PhotoCellIdentifier
                                                      forIndexPath:indexPath];
    Photo* photo = [self photoAtIndexPath:indexPath];
    cell.label.text = photo.name;
    return cell;
}
```

　　上面示例的数据源的实现都与 NSArray 有关，还有一个方法的实现与 Photo 有关(Photo 与 Cell 呈一一对应关系)。下面让我们来把与 NSArray 相关的代码从 控制器中抽离出来,并改用 block 来设置 cell 的视图。当然你也可以用代理来实现，取决于你的个人喜好。
　　
```python
@implementation ArrayDataSource

- (id)itemAtIndexPath:(NSIndexPath*)indexPath {
    return items[(NSUInteger)indexPath.row];
}

- (NSInteger)tableView:(UITableView*)tableView
 numberOfRowsInSection:(NSInteger)section {
    return items.count;
}

- (UITableViewCell*)tableView:(UITableView*)tableView
        cellForRowAtIndexPath:(NSIndexPath*)indexPath {
    id cell = [tableView dequeueReusableCellWithIdentifier:cellIdentifier
                                              forIndexPath:indexPath];
    id item = [self itemAtIndexPath:indexPath];
    configureCellBlock(cell,item);
    return cell;
}

@end
```

　　现在我们可以控制器中的三个数据源代理方法可以干掉，并且把 控制器的 dataSource 设置为 ArrayDataSource 的实例。
　　
```python
void (^configureCell)(PhotoCell*, Photo*) = ^(PhotoCell* cell, Photo* photo) {
cell.label.text = photo.name;
};

photosArrayDataSource = [[ArrayDataSource alloc] initWithItems:photos
cellIdentifier:PhotoCellIdentifier
configureCellBlock:configureCell];
self.tableView.dataSource = photosArrayDataSource;
```

　　通过上面的方法，你就可以把设置 Cell 视图的工作从 控制器中抽离出来。现在你不需要再关心indexPath如何与 NSArrary 中的元素如何关联，当你需要将数组中的元素在其它 UITableView 中展示时你可以重用以上代码。你也可以在 ArrayDataSource 中实现更多的方法，比如`tableView:commitEditingStyle:forRowAtIndexPath:`。
　　这样做还能带来额外的好处，我们还可以针对这部分实现编写单独的单元测试。不仅仅针对NSArray，我们可以使用这种分离思路处理其他数据容器（比如`NSDictionary`）。
　　该技巧同样适用于其他 Protocol ,比如 `UICollectionViewDataSource` 。通过该协议，你可以定义出各种各样的 UICollectionViewCell 。假如有一天，你需要在代码在使用到 UICollectionView 来替代当前的 UITableView，你只需要修改几行 控制器中的代码即可完成替换。你甚至能够让你的 DataSource 类同时实现 `UICollectionViewDataSource` 协议和` UITableViewDataSource `协议。
 
 
 
**把业务逻辑移至 Model**

　　下面是一段位于 控制器中的代码，作用是找出针对用户active priority的一个列表。
　　
```python
//ViewController.m
- (void)loadPriorities {
  NSDate* now = [NSDate date];
  NSString* formatString = @"startDate <= %@ AND endDate >= %@";
  NSPredicate* predicate = [NSPredicate predicateWithFormat:formatString, now, now];
  NSSet* priorities = [self.user.priorities filteredSetUsingPredicate:predicate];
  self.priorities = [priorities allObjects];
}
```
 
　　然而，假如你把代码实现移至 User 的 Category 中，控制器中的代码将会更简洁、更清晰。
　　
将以上代码移到User+Extension.m中 
```python
//User+Extension.m
- (NSArray*)currentPriorities {
    NSDate* now = [NSDate date];
    NSString* formatString = @"startDate <= %@ AND endDate >= %@";
    NSPredicate* predicate = [NSPredicate predicateWithFormat:formatString, now, now];
    return [[self.priorities filteredSetUsingPredicate:predicate] allObjects];
}
```
  
　　ViewController.m 中的代码可以改成这个鬼样子，是不是明显要简洁许多，可读性强很多呢。
　　
```python
//ViewController.m
- (void)loadPriorities {
    self.priorities = [self.user currentPriorities];
}
```
  
　　实际开发中，有些代码很难移至 model 对象中，但是很明显这些代码与 model 对象有关。针对这种情况，我们可以创建一个 store 类，并把相关代码迁移进去。
 
 
**创建Store类**

　　在这个示例项目工程中，我们有一段用于从本地文件加载数据并解析的代码:
```python
- (void)readArchive {
    NSBundle* bundle = [NSBundle bundleForClass:[self class]];
    NSURL *archiveURL = [bundle URLForResource:@"photodata"
                                 withExtension:@"bin"];
    NSAssert(archiveURL != nil, @"Unable to find archive in bundle.");
    NSData *data = [NSData dataWithContentsOfURL:archiveURL
                                         options:0
                                           error:NULL];
    NSKeyedUnarchiver *unarchiver = [[NSKeyedUnarchiver alloc] initForReadingWithData:data];
    _users = [unarchiver decodeObjectOfClass:[NSArray class] forKey:@"users"];
    _photos = [unarchiver decodeObjectOfClass:[NSArray class] forKey:@"photos"];
    [unarchiver finishDecoding];
}
```
  
　　*控制器*不应该负责以上的工作，控制器只要负责数据调度就可以了，数据获取的工作我们完全可以交给 *store* 对象来负责。通过将这些代码从 *控制器*中抽离出来，我们可以更容易复用、测试这些方法、同时让*控制器*变得更轻巧( *Store* 对象一般负责数据的加载、缓存、持久化。*Store* 对象也经常被称作 *Service Layer* 对象，或者 *Repository* 对象)。
　　
**将网络通信（Web Service）逻辑移至Model层**

　　这与上一个主题非常相似：别把 *Web Service* 相关的代码写在 *控制器*中，应该把这部分代码抽离出来。并通过方法的回调对数据进行处理。

　　不仅如此，你还可以把处理异常情况的工作也转交给 *Store* 对象负责。
**把视图相关的代码移至View层**
　　同样构建视图（尤其是复杂视图）的代码也不应该写在 *View Controller* （关我毛事啊，我只负责调度和通信啊）中。要么使用*Interface Builder* ，要么封装一个 *Vew* 的子类来完成这部分工作。假设现在需要实现自定义一个日期选择器。我们应该新建一个 *DatePickerView* 的子类来完成构建视图的工作，而不是把这部分工作放在 *View Controller*中完成。同样的，这将是你的代码更简洁，复用性更强。

　　除了用 *code* 的形式来实现自定义视图，你也可以使用 *Interface Builder* 来完成构建自定义视图的工作。很多人都认为 *Interface Builder* 只能用于为 *View Controller* 构建视图，其实不然，你可以通过单独的 *nib* 文件来加载在 *Interface Builder*中构建的自定义视图。在示例工程当中，我们创建了一个包含了 *Photo Cell* 视图的 *PhotoCell.xib* 文件。
 ![](http://upload-images.jianshu.io/upload_images/1112722-d945cbc94313b0fc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如图所示，我们在 *view* 中创建了属性(无需设置 *File’s Owner* 对象)并把它们与 *Interface Builder* 中的视图关联起来。这个方法同样适用于构建其它自定义视图。

**通讯**

我们在*控制器*中经常需要与其它*控制器*、*Model* 、 *View* 进行通讯。虽然这本来就是 *控制器*应该负责处理的事情，但我们依然可以用尽可能少的代码完成我们控制器的负责的工作。
现在已经有很多成熟的方案来建立 *控制器*与 *View* 的通讯(例如 *KVO* 和 *fetched results controllers*)。然而 *控制器*之间的通讯目前还没有类似的方案可以借鉴。
在实际开发中，我们经常需要把 遇到需要把*控制器*持有的一些状态信息，传递到 多个 *控制器的需求*。通常我们会将这些状态信息保存在一个对象中然后传递给其他的视图控制器。这部分的瘦身技巧比较复杂，我留在以后再专门讲解吧。

**结论**

我们已经展示了一些瘦身*控制器*的方法。作为架构师我们不可能完全照搬这些设计技巧，但我们需要清楚我们这么做的目的，我们只有一个目标:使得代码更易于维护，只要架构师在review代码时时刻关注这个目标，我们可以就可以扩展这些技巧，灵活运用到项目中。通过了解这些方法，我们能够更好的处理好复杂的视图控制器，并且让这些视图控制器的代码更整洁，更清晰。
