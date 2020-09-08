---
layout: post
title: "iOS列表UITableView提速指南"
description: ""
category: 技术
tags: []
---

##UITableview

从08年到现在开发过的iOS应用不计其数了，但是面试很多人的时候，发现依然很多同学在最基本的列表控件上懂得不够深，下面就结合各方面的资料进行再一次讲解。

我们都知道纯代码是效率最高的，但是在开发成本上已经越来越不如使用Storyboard性价比高，速度快，所以本文试图结合UIStoryboard来描述一整套方案。

###简单配置
----- 
在Storyboard中拖入UITableViewController，并且修改涂涂画画<br/>
在代码区new File生成一个基于UITableviewController的自定义类，我这里暂时取名为Home。因为主页就是一个复杂的列表的不在少数吧？呵呵。<br/>
然后在Identity insepctor里修改对应的Class name，使得代码与Storyboard产生关联。<br/>
![](http://img226.poco.cn/mypoco/myphoto/20140120/13/17456374520140120133117055.jpg) 
想要做下拉刷新嘛？系统自带了一个给你，并且可以自定义换标题哦。很多人真的不知道在哪儿选中，请看下图，先选中UITableviewController，然后在选项卡中enable这个refresh选项，就自动完成了。
对应的代码还是复制进去，就会自动触发。<br/>
![](http://img226.poco.cn/mypoco/myphoto/20140120/14/17456374520140120143455079.jpg) 

然后你需要对UITableView做一些简单的配置，首先要选中UITableView，很多人看不到选项，是因为默认关闭了。。
![](http://img226.poco.cn/mypoco/myphoto/20140120/13/17456374520140120134209098.jpg) 

下图是对UITableview的简单配置。

Content是动态列表/静态列表，如果是静态的，那你基本不用写代码就能所见即所得，譬如“设置”页面就可以套用。但是诸如微博啦，朋友圈，还是老实的用动态列表，用代码控制。<br/>
Prototype Cells指会出现的cell有几种类型。这个后面再讲。<br/>
Style效果现场试一下就看出来了<br/>
Separator指的分隔行样式。<br/>
![](http://img226.poco.cn/mypoco/myphoto/20140120/13/17456374520140120134255017.jpg) 

然后你就需要代码中做一些简单的配置了,我只列重要的，这里不是基础教程，基础的还是老实的看教科书
{% highlight objc %}
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView //几组
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section//每一组分别几行
-(CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath//每一行多高
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath//每一行分别长啥样
-(void)tableView:(UITableView *)tableView willDisplayCell:(UITableViewCell *)cell forRowAtIndexPath:(NSIndexPath *)indexPath//即将显示某行
{% endhighlight %}

### Load More加载更多

加载更多的UITableview扩展控件非常多，尝试过各种第三方完美扩展后，我觉得这点小把戏也不至于需要扩展UITableview类吧？<br/>
那就是在列表追加一个Section，放在最后面，这个Section只有一个Cell,这个自定义的cell有3种状态，这些都可以自由发挥。<br/>
每当willDisplayCell的时候，你就设置他正在loading状态，给用户造成正在加载的假象，同时触发网络请求<br/>
当有网络数据返回后，自然会insert好多内容，也轮不到这个加载更多的cell显示的地方了，自然就释放了。当然了，如果没有更多内容，也可以轻松的cellForRowAtIndexPath找到唯一的cell，设置为无更多数据等状态。<br/>

{% highlight objc %}
typedef enum{
	JWLoadMoreNormal = 0,//点击再加载
    JWLoadMoreLoading,//加载中
    JWLoadMoreDone,//无更多数据
} JWLoadMoreState;

@interface LoadMoreCell : UITableViewCell

@property (strong, nonatomic)  UIView *baseWhiteView;
@property (strong, nonatomic)  UIActivityIndicatorView *activity;
@property (strong, nonatomic)  UILabel *tipsLabel;
@property (strong, nonatomic)  UIButton *loadMoreButton;
@property (nonatomic,unsafe_unretained) id <LoadMoreCellDelegate>                   delegate;
-(void)setupUI;
- (void)loadMore:(id)sender;
-(void)setLoadMoreStatus:(JWLoadMoreState)status;

{% endhighlight %}


###Autolayout
做完这些，基本配置就完成了，下面需要根据设计师的要求进行自定义开发，譬如自定义cell
![](http://img226.poco.cn/mypoco/myphoto/20140120/14/17456374520140120143516034.jpg) 
如上图，密密麻麻的

autolayout的拖拽不会？你太老土了吧。xcode5的拖拽，可谓是异常简单，只需要点快捷菜单的pin，设置好上下左右相对关系就可以了。<br/>
创建custom的uitableviewcell基本差不多，拖出来，画画涂涂，创建代码，改类名对应关系，按着“Control"拖拽关联，等等。<br/>

我这里只讲一个特殊的，就是图上“图片”“摘要”属于并排区域，可能没有图片的帖子，“摘要”就需要顶格排版。这样的情况该怎么设置呢？<br/>
这就得用NSLayoutConstraint的拽出来的关联了。把”摘要“的相对距离，锁定在一个固定的位置上，譬如”左边栏“，通过少量的代码计算，即可动态的修改NSLayoutConstraint.constant的距离。<br/>

{% highlight objc %}
    NSString *url=data.img;
    [self.previewImageView setClipsToBounds:YES];
    if ( url!=nil && ![url isEqualToString:@""]) {  //这里是判断有图没图
        [self.previewImageView setImageWithURL:[NSURL URLWithString:url] placeholderImage:[UIImage imageNamed:@"pic_default"]];
        self.previewImageHeight.constant=HomeCellImageHeight;
        self.previewImageWidth.constant=HomeCellImageWidth;
        self.abstractLeftMargin.constant=10;  //若有图，摘要与图的左边距变为10
        self.summaryDistanceBetweenTitle.constant=80;
    }
    else
    {
        [self.previewImageView setImage:nil];
        self.previewImageHeight.constant=0;  //图片大小全清空为0
        self.previewImageWidth.constant=0;
        self.abstractLeftMargin.constant=0;  //若无图，摘要与图的左边距变为10
    }
    [self setNeedsUpdateConstraints];   //记得强制刷新，要不然系统懒懒的
    [self.contentView layoutIfNeeded];
    [self.contentView setNeedsLayout];
{% endhighlight %}

###动态计算高度


做到这里，恐怕大部分人都遇到一个门槛了。那就是如何动态计算cell的高度。<br/>
最简单的，就是网易新闻类，固定高度。return 44;<br/>
若是动态的，无非是创建一个cell，并且初始化构造好，然后输出cell的最后一行控件的位置，最终给出位置。<br/>

但是这就导致了函数运行的低效。你想，autolayout本来就够效率低了（因为程序猿省事儿了），再为每一行计算2次，这效率能高？<br/>
我亲测发现，动态排版效率是非常低的，不足以信任。<br/>

最好办的，还是土方法。获取对应的数据，根据自己设定的排版规则，动态的计算。土归土，效率高啊！
{% highlight objc %}

        TopicID *data=[threadList objectAtIndex:indexPath.row];
//        NSLog(@"计算行数为%d",indexPath.row);
        int height=0;
        height+=17; //用户名与顶部空间
        height+=17; //用户名高度
        height+=8; //用户名与标题之间距离
        
        NSString *titleContent=data.title;
        UIFont *titleFont = [UIFont boldSystemFontOfSize:16.0];
        CGSize titleRealSize=[titleContent sizeWithFont:titleFont constrainedToSize:CGSizeMake(280, 150) lineBreakMode:NSLineBreakByTruncatingTail];
        height+=titleRealSize.height;//动态计算标题高度
        
        if (data.img!=nil && ![data.img isEqualToString:@""]) {
            height+=HomeCellImageHeight;//统计图的起始点
            height+=20;
//            NSLog(@"有图");
        }
        else
        {
            NSString *abstract=data.abstract;
            UIFont *abstractFont = [UIFont boldSystemFontOfSize:14.0];
            CGSize abstractRealSize=[abstract sizeWithFont:abstractFont constrainedToSize:CGSizeMake(280, 300) lineBreakMode:NSLineBreakByTruncatingTail];
            
            height+=abstractRealSize.height;
            height+=20;
//            NSLog(@"无图");
        }
        height+=56; //统计图的高度
        height+=3;//下边框的高度

{% endhighlight %}


最后别忘了Profile，计算时间，每一个细节的时间优化，最终都会体现在列表的流畅度表现上。
通过以上几点呢，再结合现成好用的SDWebImageCache，相信大家一定可以做出真正美观、高效的列表哦！







