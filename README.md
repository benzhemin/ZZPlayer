# ZZPlayer
# Swift AVPlayer 播放网络视频之基础篇

http://ios.jobbole.com/88185/


首页所有文章业界动态iOS开发Swift开发产品推广产品设计iOS小组更多频道 ▼
伯乐在线 > IOS - 伯乐在线 > 所有文章 > iOS开发 > Swift AVPlayer 播放网络视频之基础篇
Swift AVPlayer 播放网络视频之基础篇

2016/08/21 · iOS开发 · AVPlayer
分享到： 1
原文出处： 大石头布（@大石头布）   
11954071-f0f25ee81a868945
本篇效果图

效果
iOS 播放视频可以用MPMoviePlayerController，MPMoviePlayerController是系统高度封装的VC，使用起来相对简单，但是灵活性缺失，
一般播放视频会选择使用AVPlayer ， 它可以高度自定义。虽说 AVPlayer 也有一些库，如果项目比较紧可以用第三方，但是如果有时间还是要自己学习的。本篇以AVPlayer播放网络视频为例，介绍 AVPlayer的基本用法。后面慢慢重构，尽量写出一个功能强大的播放器。

准备工作

找到Info.plist 右键 Open As -> Source Code , 在 上面加上：

Objective-C

1
2
3
4
5
<key>NSAppTransportSecurity</key>
    <dict>
        <key>NSAllowsArbitraryLoads</key>
        <true/>
    </dict>
不加的不能处理http请求。都懂的。

加载出视频

首先来介绍用到的几个对象：

AVPlayerItem 一个媒体资源管理对象，管理者视频的一些基本信息和状态，如 播放进度、缓存进度等 。 一个AVPlayerItem对应着一个视频资源。
AVPlayer 视频操作对象，但是无法显示视频，需要把自己添加到一个AVPlayerLayer 上
AVPlayerLayer 用来显示视频的
我们先来自定义一个 ZZPlayerView 继承自 UIView ， 这个 View 是用来显示视频和处理一些基本操作的。

1、添加一个变量 var playerLayer:AVPlayerLayer?
2、在layoutSubviews 方法中指定layer的大小

Objective-C

1
2
3
4
override func layoutSubviews() {
    super.layoutSubviews()
    playerLayer?.frame = self.bounds
}
然后在Main.stroyboard 中拖一个UIView 到VC上 ， 把 Class 设置成ZZPlayerView .

然后在VC中声明几个对象 :

Objective-C

1
2
3
4
5
@IBOutlet weak var playerView:ZZPlayerView!
 
var playerItem:AVPlayerItem!
var avplayer:AVPlayer!  
var playerLayer:AVPlayerLayer!
上面提到的三个对象和自定义的View 。

然后ViewDidLoad中 添加如下代码 ：

Objective-C

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
// 检测连接是否存在 不存在报错
guard let url = NSURL(string: "http://bos.nj.bpc.baidu.com/tieba-smallvideo/11772_3c435014fb2dd9a5fd56a57cc369f6a0.mp4") else { fatalError("连接错误") }
 
playerItem = AVPlayerItem(URL: url) // 创建视频资源 
// 监听缓冲进度改变
playerItem.addObserver(self, forKeyPath: "loadedTimeRanges", options: NSKeyValueObservingOptions.New, context: nil)
// 监听状态改变
playerItem.addObserver(self, forKeyPath: "status", options: NSKeyValueObservingOptions.New, context: nil)
// 将视频资源赋值给视频播放对象
self.avplayer = AVPlayer(playerItem: playerItem)
// 初始化视频显示layer
playerLayer = AVPlayerLayer(player: avplayer)
// 设置显示模式
playerLayer.videoGravity = AVLayerVideoGravityResizeAspect
playerLayer.contentsScale = UIScreen.mainScreen().scale
// 赋值给自定义的View
self.playerView.playerLayer = self.playerLayer
// 位置放在最底下
self.playerView.layer.insertSublayer(playerLayer, atIndex: 0)
上面加了监听记得在页面销毁的时候remove掉 :

Objective-C

1
2
3
4
deinit{
    playerItem.removeObserver(self, forKeyPath: "loadedTimeRanges")
    playerItem.removeObserver(self, forKeyPath: "status")
}
然后处理监听事件：

Objective-C

1
2
3
4
5
6
7
8
9
10
11
12
13
14
override func observeValueForKeyPath(keyPath: String?, ofObject object: AnyObject?, change: [String : AnyObject]?, context: UnsafeMutablePointer) {
    guard let playerItem = object as? AVPlayerItem else { return }
    if keyPath == "loadedTimeRanges"{
        // 缓冲进度 暂时不处理
    }else if keyPath == "status"{
        // 监听状态改变 
        if playerItem.status == AVPlayerItemStatus.ReadyToPlay{
            // 只有在这个状态下才能播放
            self.avplayer.play()
        }else{
            print("加载异常")
        }
    }
}
一共有三种状态 Unknown 、ReadyToPlay 、 Failed 只有在 ReadyToPlay 状态下视频才能播放。
好了 运行视频，如果一切正常的话（一般没问题）。视频可以正常播放出来的。网络加载的，网速慢的 等等。

时间显示

没见过哪个视频播放器是光秃秃的，啥都没有， 没有进度条 、没有时间显示 。我们先给播放器加上时间显示 。
我们的布局最好使用AutoLayout , 因为视频一般是支持横竖屏的，AutoLayout 会省去你很多麻烦， 代码写Autolayout 比较麻烦，所以这里选用了SnapKit
, 一个AutoLayout 的库， 使AutoLayout在代码中的使用语法异常简洁.为了简单，我直接将源码拖到工程目录下使用了。下载地址 ： SnapKit

因为我们这里使用了storyboard 创建的ZZPlayerView , 所以要手动添加View的话，需要在 awakeFromNib方法中处理

在ZZPlayerView 中声明变量 var timeLabel:UILabel! .
然后在 awakeFromNib 中布局它的位置。

Objective-C

1
2
3
4
5
6
7
8
timeLabel = UILabel()
timeLabel.textColor = UIColor.whiteColor()
timeLabel.font = UIFont.systemFontOfSize(12)
self.addSubview(timeLabel)
timeLabel.snp_makeConstraints { (make) in
    make.right.equalTo(self)
    make.bottom.equalTo(self).inset(5)
}
然后回到ViewController中，写一个将秒转成时间字符串的方法，因为我们将得到秒。

Objective-C

1
2
3
4
5
6
7
8
func formatPlayTime(secounds:NSTimeInterval)->String{
    if secounds.isNaN{
        return "00:00"
    }
    let Min = Int(secounds / 60)
    let Sec = Int(secounds % 60)
    return String(format: "%02d:%02d", Min, Sec)
}
因为在刚开始的时候我们可能得到的不是一个数字，所以加上了判断 。

因为我们要实时计算时间，这里加上一个计时器。一般会选择NSTimer , 但是这里我们选择CADisplayLink .
声明一个变量 var link:CADisplayLink!

在viewDidLoad最底下加两句话 。

Objective-C

1
2
self.link = CADisplayLink(target: self, selector: #selector(update))
self.link.addToRunLoop( NSRunLoop.mainRunLoop(), forMode: NSDefaultRunLoopMode)
CADisplayLink 的执行次数相当于屏幕的帧数，iPhone 不卡顿的时候是每秒60次。把它加入主loop中，默认Mode 。 关于NSRunLoop 又是一个大话题，感兴趣的，以后一起讨论。这里先这么写着。差不多每秒执行60次。

然后我们来处理这个update 方法

Objective-C

1
2
3
4
5
6
7
8
9
10
func update(){
    // 当前播放到的时间
    let currentTime = CMTimeGetSeconds(self.avplayer.currentTime())
    // 总时间
    let totalTime   = NSTimeInterval(playerItem.duration.value) / NSTimeInterval(playerItem.duration.timescale)
    // timescale 这里表示压缩比例
    let timeStr = "\(formatPlayTime(currentTime))/\(formatPlayTime(totalTime))" // 拼接字符串
    playerView.timeLabel.text = timeStr // 赋值
    // TODO: 播放进度 
}
现在执行。是不是可以实时显示时间了。

处理进度条

进度条我们使用UISlider + UIProgressView的方式。

首先在ZZPlayerView添加一个UISlider。声明变量var slider:UISlider!。
awakeFromNib 中加入以下代码：

Objective-C

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
slider = UISlider()
self.addSubview(slider)
slider.snp_makeConstraints { (make) in
    make.bottom.equalTo(self).inset(5)
    make.left.equalTo(self).offset(50)
    make.right.equalTo(self).inset(100)
    make.height.equalTo(15)
}
slider.minimumValue = 0
slider.maximumValue = 1
slider.value = 0
// 从最大值滑向最小值时杆的颜色
slider.maximumTrackTintColor = UIColor.clearColor()
// 从最小值滑向最大值时杆的颜色
slider.minimumTrackTintColor = UIColor.whiteColor()
// 在滑块圆按钮添加图片
slider.setThumbImage(UIImage(named:"slider_thumb"), forState: UIControlState.Normal)
我们指定了slider的位置，并指定了最大值和最小值。当前值，还有更换了滑块的图片，系统的太丑了。

然后在update方法后面加上下面一句话，就可以看到播放进度了 。

Objective-C

1
self.playerView.slider.value = Float(currentTime/totalTime)
运行下， 不错确实有效果。 而且滑块还不错 。（ 如果你替换了个不错的图片的话 ，你也可以下载我的项目，使用我的图片。我这边图片是用Sketch 随便画的 ）

这时候手动滑动滑块并没有用，会立刻回去。我们还需要处理滑块事件，改变视频播放进度。只需要处理两个事件

在ZZPlayerView的awakeFromNib最底下加上以下代码：

Objective-C

1
2
3
4
5
6
// 按下的时候
slider.addTarget(self, action: #selector(sliderTouchDown( _:)), forControlEvents: UIControlEvents.TouchDown)
// 弹起的时候
slider.addTarget(self, action: #selector(sliderTouchUpOut( _:)), forControlEvents: UIControlEvents.TouchUpOutside)
slider.addTarget(self, action: #selector(sliderTouchUpOut( _:)), forControlEvents: UIControlEvents.TouchUpInside)
slider.addTarget(self, action: #selector(sliderTouchUpOut( _:)), forControlEvents: UIControlEvents.TouchCancel)
为了保险，我们在弹起的时候监听了三个方法。

因为我们不希望我们在滑动的时候还一直改变播放进度，所以在ZZPlayerView 加上变量var sliding = false 表示是否正在滑动

然后处理按下和弹起事件：

Objective-C

1
2
3
4
5
6
7
func sliderTouchDown(slider:UISlider){
    self.sliding = true
}
func sliderTouchUpOut(slider:UISlider){
    // TODO: -代理处理
 
}
在update 方法中修改处理播放进度的逻辑

Objective-C

1
2
3
4
5
// 滑动不在滑动的时候
if !self.playerView.sliding{
    // 播放进度
    self.playerView.slider.value = Float(currentTime/totalTime)
}
滑动结束的时候需要改变视频进度，所以这里写一个代理。

Objective-C

1
2
3
protocol ZZPlayerViewDelegate:NSObjectProtocol {
    func zzplayer(playerView:ZZPlayerView,sliderTouchUpOut slider:UISlider)
}
ZZPlayerView 中声明 weak var delegate:ZZPlayerViewDelegate?

然后弹起事件中加入：

Objective-C

1
delegate?.zzplayer(self, sliderTouchUpOut: slider)
然后在VC中实现代理：（别忘记在viewDidLoad中加上self.playerView.delegate = self）

Objective-C

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
extension ViewController:ZZPlayerViewDelegate{
    // 滑动滑块 指定播放位置
    func zzplayer(playerView: ZZPlayerView, sliderTouchUpOut slider: UISlider) {
 
        //当视频状态为AVPlayerStatusReadyToPlay时才处理
        if self.avplayer.status == AVPlayerStatus.ReadyToPlay{
            let duration = slider.value * Float(CMTimeGetSeconds(self.avplayer.currentItem!.duration))
            let seekTime = CMTimeMake(Int64(duration), 1)
            // 指定视频位置
            self.avplayer.seekToTime(seekTime, completionHandler: { (b) in
                // 别忘记改状态
                playerView.sliding = false
            })
        }
    }
}
这时候运行已经可以根据滑块改变进度了，但是我们一直忘记处理缓存进度了。

在ZZPlayerView 中加一个变量 var progressView:UIProgressView!。

还是在awakeFromNib 中进行布局。

Objective-C

1
2
3
4
5
6
7
8
9
10
11
progressView = UIProgressView()
progressView.backgroundColor = UIColor.lightGrayColor()
self.insertSubview(progressView, belowSubview: slider)
progressView.snp_makeConstraints { (make) in
    make.left.right.equalTo(slider)
    make.centerY.equalTo(slider)
    make.height.equalTo(2)
}
 
progressView.tintColor = UIColor.redColor()
progressView.progress = 0
为了比较清晰的看到进度，我们这里设置了红色。进度需要显示在slider下面和slider位置一样。

我VC中写一个方法来计算当前的缓冲进度

Objective-C

1
2
3
4
5
6
7
8
func avalableDurationWithplayerItem()->NSTimeInterval{
    guard let loadedTimeRanges = avplayer?.currentItem?.loadedTimeRanges,first = loadedTimeRanges.first else {fatalError()}
    let timeRange = first.CMTimeRangeValue
    let startSeconds = CMTimeGetSeconds(timeRange.start)
    let durationSecound = CMTimeGetSeconds(timeRange.duration)
    let result = startSeconds + durationSecound
    return result
}
然后在我们前面写的TODO也就是KVO监听缓冲进度的地方换成下面代码 :

Objective-C

1
2
3
4
5
6
7
8
if keyPath == "loadedTimeRanges"{
    // 通过监听AVPlayerItem的"loadedTimeRanges"，可以实时知道当前视频的进度缓冲
    let loadedTime = avalableDurationWithplayerItem()
    let totalTime = CMTimeGetSeconds(playerItem.duration)
    let percent = loadedTime/totalTime // 计算出比例
    // 改变进度条
    self.playerView.progressView.progress = Float(percent)
}
至此，我们已经很好的处理了进度相关的工作 。 但是我们视频缺少播放和暂停键 。

播放-暂停

废话不多，继续干活。

ZZPlayerView 中添加var playBtn:UIButton!变量。
awakeFromNib中进行布局。

Objective-C

1
2
3
4
5
6
7
8
9
10
11
 playBtn = UIButton()
self.addSubview(playBtn)
playBtn.snp_makeConstraints { (make) in
    make.centerY.equalTo(slider)
    make.left.equalTo(self).offset(10)
    make.width.height.equalTo(30)
}
// 设置按钮图片
playBtn.setImage(UIImage(named: "player_pause"), forState: UIControlState.Normal)
// 点击事件
playBtn.addTarget(self, action: #selector(playAndPause( _:)) , forControlEvents: UIControlEvents.TouchUpInside)
这里又会设置一个状态表示，是否在播放，用来切换按钮的图片。播放和暂停是不同图片（同样可以下载我的项目使用，最后下载项目一期看）

添加变量var playing = true 表示是否正在播放 ， 因为播放和暂停是在VC中处理，这里依旧是代理。

协议中添加方法：

Objective-C

1
func zzplayer(playerView:ZZPlayerView,playAndPause playBtn:UIButton)
处理点击事件

Objective-C

1
2
3
4
5
6
7
8
9
10
11
12
13
14
func playAndPause(btn:UIButton){
    let tmp = !playing
    playing = tmp // 改变状态
 
    // 根据状态设定图片
    if playing {
        playBtn.setImage(UIImage(named: "player_pause"), forState: UIControlState.Normal)
    }else{
        playBtn.setImage(UIImage(named: "player_play"), forState: UIControlState.Normal)
    }
 
    // 代理方法    
    delegate?.zzplayer(self, playAndPause: btn)
}
在VC中加上：

Objective-C

1
2
3
4
5
6
7
8
9
func zzplayer(playerView: ZZPlayerView, playAndPause playBtn: UIButton) {
    if !playerView.playing{
        self.avplayer.pause()
    }else{
        if self.avplayer.status == AVPlayerStatus.ReadyToPlay{
            self.avplayer.play()
        }
    }
 }
然后在update的最前面加上

Objective-C

1
2
3
4
//暂停的时候
if !self.playerView.playing{
    return
}
因为在暂停的时候不需要计算。

至此，我们基础版本的播放器实现完了，效果如图。这个播放器还缺少很多功能，比如：右划快进、左划快退 ，播放速度 1x 2x 0.5x等 左边上滑调整亮度 右边上滑调整音量 、 清晰度切换、 缓存下载等 。 而且还需要进一步封装。有时间会把这些补上 。

github链接:https://github.com/smalldu/ZZPlayer
blog同步地址:Swift AVPlayer 播放网络视频之基础篇
参考链接:
AVPlayer 本地、网络视频播放相关
BMPlayer
