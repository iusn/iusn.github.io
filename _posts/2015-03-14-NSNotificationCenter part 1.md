---

layout: post
title:  "NSNotificationCenter part 1: Receiving and sending notifications"
date: 2015-03-14
description: ""
category: translation
tags: [NSNotificationCenter]
comments: true
share: true

---


 
新手第一次翻译如果有哪里不对的地方希望您提出^_^


##[原文](http://www.hpique.com/2013/12/nsnotificationcenter-part-1/)

NSnotificationCenter经常被忽视，也许是因为和争议颇多的KOV搭配在一起，或者是因为代理模式如此的普遍。然而当它要给一些对象发送广播时，它能使用最少的方法和类实现所有的广播。如果你曾经你发现用观察者模式通过代理通知的事件要考虑如何避开retain cycles ，或者用blocks处理用户正在离开一个界面然后又创建个新的
UIviewController实例，用notifications就会有一个更加清晰的解耦解决方案。

这篇文章是NSNotificationCenter系列的第一篇。

- [part1](http://www.hpique.com/2013/12/nsnotificationcenter-part-2/)涵盖了接收和发送通知和常见的误解。
- [part2](http://www.hpique.com/2013/12/nsnotificationcenter-part-3/)用notifications的观察者模式。
- [part3]()如何用OCMock实现notifications的单元测试。
- [part4](http://www.hpique.com/2013/12/nsnotificationcenter-part-4/)用NSNotificationQueue处理异步通知。
	
最后[iOS](https://gist.github.com/hpique/7554209)和[OS X](https://gist.github.com/hpique/8198196)的一系列的公共通知都在附件里。我们将用iOS的类举例子，在OS X中也实用。

##Notifications类
--

Foundation框架提供了3个类供你处理通知

- NSNotification: 担任通知角色
- NSNotificationCenter: 广播通知管理观察者。每个app有一个默认的通知中心defaultCenter
- NSNotificationQueue: 合并和推迟发送通知。通过defaultQueue方法每个线程都有一个默认的通知队列。

OS X 为了在两个进程之间通信提供第四个方法叫NSDistributedNotificationCenter，在这个系列里我们将不提它。


##Receiving notifications
--

要接收一个通知你只需要知道他的名字就行了。Cocoa和Cocoa Touch框架对消息的命名很有趣比如 UIKeyboardWillShowNotification and UIApplicationDidReceiveMemoryWarningNotification 在iOS7（[annex A](https://gist.github.com/hpique/7554209)）和OS X10.9([annex B](https://gist.github.com/hpique/8198196))中分别有165和393个公共通知，之后我们来学习如何创建自己的notifications。

接收一个通知只需要很少的一段代码。比如你想在UIViewController之外收到一个memory warning(UIApplicationDidReceiveMemoryWarningNotification)之后清除缓存对象，代码应该是这样的


{% highlight objective-c %}
-(id) init {
    if (self = [super init]) {
        _cache = [[NSCache alloc] init];
        [[NSNotificationCenter defaultCenter] addObserver:self
            selector:@selector(applicationDidReceiveMemoryWarning:)
            name:UIApplicationDidReceiveMemoryWarningNotification object:nil];
    }
    return self;
}
 
-(void) applicationDidReceiveMemoryWarning:(NSNotification*)notification {
    [_cache removeAllObjects];
}
 
-(void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self
        name:UIApplicationDidReceiveMemoryWarningNotification];
}
{% endhighlight %}

 
 
##Adding the observer
--

接收通知的对象被称为观察者，必须加到NSNotificationCenter。除非你有很强的理由不这么做，否则永远都是defaultCenter。

在UIviewController中把观察者添加到init,viewDidLoad和viewWillAppear， 都是很好的选择。为了提高性能和避免一些不好的副作用，你应该尽可能晚的添加观察者然后再尽可能早的删除它。 

通过指定通知的名字，观察者可以注册一个相应的通知，这相当于过滤器的作用。在上面的例子里我们不关心谁发送的消息所以设置为nil。

当不同实例发送相同名字的通知指定发送者的名字就很有用哦（比如你可能对UITextFieldTextDidChangeNotification中特定的UITextField这个通知感兴趣）。如果没有指定通知的发送者，你就会收到所有被指定了这个相同名字的通知。如果没有指定通知的名字你就会收到所有的发送者发送的通知。

##Handling the notification
--

观察者接收通知直到被从NSNotificationCenter中移除。每次调用addObserver通知都会被发送一次。如果你接收了太多的通知，有可能添加了不止一次观察者（比如你再viewWillAppear中注册了观察者但是没有在dealloc中销毁。）

NSNotificationCenter 会在addObserver:selector:name:object: 中使用selector选择器提供一个NSNotification类型的参数。这个选择器selector会在通知被post的同一线程中调用。除非在相应的文档说明这个选择器selector运行在不同的线程中，否则一般都是在主线程中。

NSNotification对象是一个集合。它有一个对象id  这个对象id一般是消息的发送者和userInfo字典关于通知的附加信息。比如：


{% highlight objective-c %}
-(void) moviePlayerPlaybackDidFinish:(NSNotification*)notification {
    MPMoviePlayerController *mpObject = (MPMoviePlayerController *) notification.object; 
    NSDictionary *userInfo = notification userInfo;
    MPMovieFinishReason reason = [[userInfo objectForKey:MPMoviePlayerPlaybackDidFinishReasonUserInfoKey] intValue];
 }
{% endhighlight %}


 想知道字典中制定的通知需要debug或者查文档。在没有被告知的情况下不要假设任何一个key。
 
你也可以用blocks处理通知来做单元测试
 

{% highlight objective-c %}
-(id) init {
    if (self = [super init]) {
        _cache = [[NSCache alloc] init];
        _observer = [[NSNotificationCenter defaultCenter]
            addObserverForName:UIApplicationDidReceiveMemoryWarningNotification 
            object:nil
            queue:nil
            usingBlock:^(NSNotification *notification) {
                [_cache removeAllObjects];
            }
        ];
    }
    return self;
}
 
-(void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:_observer
        name:UIApplicationDidReceiveMemoryWarningNotification];
}
{% endhighlight %}



队列中的参数是为异步通知准备的在[part4](http://www.hpique.com/2013/12/nsnotificationcenter-part-4/)中有解释


##Removing the observer
--

你在以后再也不需要通知时观察者应该立即被移除。最好的时机就是在接受到最后一条通知(MPMoviePlayerPlaybackDidFinishNotification之后加载UI)最坏的时机就是在dealloc方法里。如果你忘了注销观察者NSNotificationCenter 有可能把通知发送给一个已注销了的实例。

NSNotificationCenter 提供两个注销观察者的方法。removeObserver: 和removeObserver:name:object:。前者是注销所有的观察者，后者指定了被注销者的名字和发送者，名字和发送者可以是nil。虽然很容易调用removeObserver：但是如果你在非注册的地方注销掉，这是一个不好的习惯（比如父类，子类，类别中），所以当注销时尽量指定名字和发送者。

你也可以像上面那样用blocks注销观察者，在这里观察者在addObserverForName:object:queue:usingBlock:中添加，对于这些block观察者你使用单一参数的removeObserver：注销会更好。

##Sending synchronous notifications
--
通知对于广播事件很友好，比如模型的改变，用户动作，后台程序的状态。与委托模式不同的是，notifications可以用于通知一些相关的对象或者添加一些低耦合的事件（在大多数情况下，观察者只需要知道通知的名字）

##Naming notifications
--
创建自己的通知的第一步就是要选择一个唯一的名字。文档[Coding Guidelines for Cocoa](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CodingGuidelines/CodingGuidelines.html)中建议要这样：

{% highlight objective-c %}
[Name of associated class] + [Did | Will] + [UniquePartOfName] + Notification
{% endhighlight %}

比如：

@"HPEAudioPlayerDidStartPlaybackNotification" 这样就是很好的命名。

@"didStartPlayback" 这个命名不好
		
通知的命名必须是公共的常量，这样在观察者中你就可以只改通知的名字不需要改变其他的。

##Posting the notification

每个应用中都有被用来广播通知的默认NSNotificationCenter实例。你可以创建通知对象并post。如果你发送一个不需要发送附加的信息的通知很简单比如：

{% highlight objective-c %}
[[NSNotificationCenter defaultCenter] postNotificationName:notificationName object:notificationSender];
{% endhighlight %}


这个object参数通常用于传递对象发送的通知，正常情况下self. NSNotificationCenter 将会用给定的名字和发送者创建一个没有附加信息的NSNotification对象

通知被无序的发送到观察者哪里。发送同步通知 意味着直到所有的观察者处理完通知否则postNotification:不会返回。另外观察者将在通知被发送的相同线程中调用。除非你显式的声明，否则观察者认为在主线程中接收通知

考虑到要在后台定期更新进度。我们想要一边发送通知一边进行我们的操作还要在主线程中通知观察者。用调度队列实现的方法是：

{% highlight objective-c %}
while (!finished) {
    // Do something
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter]
            postNotificationName:HPEBackgroundOperationDidIncreaseProgressNotification
            object:self];
    });
}
{% endhighlight %}


##Sending additional information

在大多数情况你想要在通知中发送附加信息，比如上面的进度值。通知有一个带有自定义值的字典叫userInfo。NSNotifcationCneter 提供一个方便的方法发送带有userInfo信息的通知

{% highlight objective-c %}
NSDictionary *userInfo = @{@"someKey": someValue};
[[NSNotificationCenter defaultCenter] postNotificationName:notificationName object:notificationSender userInfo:userInfo];
{% endhighlight %}


再一次声明 你必须定义公共常量作为userInfo 的key。
 
##Subclassing NSNotification
 
如果userInfo字典不符合你的需求，或者你喜欢定义属性类型你可以子类化NSNotification。这里有一个小小的警告
 
NSNotification 是Cocoa的类簇，类簇就是以另一种方式调用抽象类。特别是没有实现name，object，userInfo和调用init抛出异常。
因为有了上面的警告，子类化NSNotification 一般是不赞成的并且一些开发人员喜欢使用类别，这两个例子在part2中提供。
 
 当你使用你自己的NSNotification的子类时只是简单的发送通知不是很好。你必须在子类的implementation中或者调用postNotification：之前创建一个通知对象并提供它的name，object如果合适的话还要提供userInfo。比如：


{% highlight objective-c %}
 HPECustomNotification *notification = [[HPECustomNotification alloc] initWithObject:self];
[[NSNotificationCenter defaultCenter] postNotification:notification];
{% endhighlight %}


##Further reading

- [Part 1: Receiving and testing notifications](http://www.hpique.com/2013/12/nsnotificationcenter-part-1/)
