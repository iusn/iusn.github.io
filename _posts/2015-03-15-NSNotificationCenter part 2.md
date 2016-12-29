---

layout: post
title:  "NSNotificationCenter part 2: Implementing the observer pattern with notifications"
date: 2015-03-15
description: ""
category: translation
tags: [NSNotificationCenter]
comments: true
share: true

---


##[原文](http://www.hpique.com/2013/12/nsnotificationcenter-part-2/)
--

这篇文章是NSNotificationCenter 系列的第二篇。[第一篇](http://iusn.github.io/NSNotificationCenter%20part%201/)涵盖了接收和发送通知

如果你曾尝试用Objective-C实现过观察者模式，你可能会注意到需要使用非固定的集合来避免retain cycles。

为了不使用自定义的观察者，一个更简洁的方法是使用NSNotificationCenter。但是我们要是直接使用它，就会丢失观察者模式的强类型。下面就是如何使用NSNotificationCenter实现这种流行的模式。


<!--more-->


##Typed observer
--
首先，定义观察者协议，在这个例子中我们自定义一个 audio player(HPEAudioPlayer)


{% highlight objective-c %}
@protocol HPEAudioPlayerObserver& lt;NSObject& gt;
@optional

-(void)audioPlayerDidStartPlayback:(NSNotification*)notification;

-(void)audioPlayerDidFinishPlayback:(NSNotification*)notification;

@end
{% endhighlight %}

每个observer方法对应一个通知，因此只有一个单一的notificaion参数。observer不需要知道通知的名字，这是一个实现的细节。

然后observable类中添加方法来现实添加和删除观察者

{% highlight objective-c %}
@interface HPEAudioPlayer(Observable)

-(void)addObserver:(id& lt;HPEAudioPlayerObserver& gt;)observer;
-(void)removeObserver:(id& lt;HPEAudioPlayerObserver& gt;)observer;

@end
{% endhighlight %}

在observable类的实现方法中我们定义通知名字为常量

{% highlight objective-c %}
NSString *const HPEAudioPlayerDidStartPlaybackNotificationName = @"HPEAudioPlayerDidStartPlaybackNotification";

NSString *const HPEAudioPlayerDidFinishPlaybackNotificationName = @"HPEAudioPlayerDidFinishPlaybackNotification"
{% endhighlight %}

使用适当的选择器添加或者删除观察者以减少对NSNotificationCenter调用

{% highlight objective-c %}
-(void)addObserver:(id& lt;HPEAudioPlayerObserver& gt;)observer {
    { // audioPlayerDidStartPlayback
        SEL selector = @selector(audioPlayerDidStartPlayback:);
        if ([observer respondsToSelector:selector]) {
            [[NSNotificationCenter defaultCenter] addObserver:observer selector:selector name:HPEAudioPlayerDidStartPlaybackNotificationName object:self];
        }
    }
    { // audioPlayerDidFinishPlayback
        SEL selector = @selector(audioPlayerDidFinishPlayback:);
        if ([observer respondsToSelector:selector]) {
            [[NSNotificationCenter defaultCenter] addObserver:observer selector:selector name:HPEAudioPlayerDidFinishPlaybackNotificationName object:self];
        }
    }
}

-(void)removeObserver:(id& lt;HPEAudioPlayerObserver& gt;)observer {
    [[NSNotificationCenter defaultCenter] removeObserver:observer name:HPEAudioPlayerDidStartPlaybackNotificationName];
    [[NSNotificationCenter defaultCenter] removeObserver:observer name:HPEAudioPlayerDidFinishPlaybackNotificationName];
}
{% endhighlight %}


最后通知观察者就像发送对应的通知一样。接下来的就交给NSNotificationCenter

{% highlight objective-c %}
-(void)notifyDidStartPlayback {
    [[NSNotificationCenter defaultCenter] postNotificationName:HPEAudioPlayerDidStartPlaybackNotificationName object:self]
}

-(void)notifyDidFinishPlayback {
    NSDictionary *userInfo = @{HPEAudioPlayerNotificationEndTimeKey: @(_endTime)};
    [[NSNotificationCenter defaultCenter] postNotificationName:HPEAudioPlayerDidFinishPlaybackNotificationName object:self userInfo:userInfo]
}
{% endhighlight %}

##Typed parameters with categories
---

当使用观察者模式时，通知方法要有一个好的参数类型，可怕的是在上面的例子中我们使用通用的NSNotification类型作为我们唯一的参数的类型。object和userInfo呢？别急我们可以在NSNotification类中使用类别来做的稍微好一点。

对于我们audio player例子我们可以使用类别来实现

{% highlight objective-c %}
@interface NSNotification (HPEAudioPlayerObserver)

@property (nonatomic, readonly) HPEAudioPlayer *hpe_audioPlayer;
@property (nonatomic, readonly) NSTimeInterval hpe_endTime;

@end

@implementation NSNotification (HPEAudioPlayerObserver)

-(HPEAudioPlayer*)hpe_audioPlayer {
    return self.object;
}

-(NSTimeInterval)hpe_endTime {
    return [[self.userInfo objectForKey:HPEAudioPlayerNotificationEndTimeKey] doubleValue];
}

@end

{% endhighlight %}

通过使用类别方法发送者不需要指定一个object也不需要userInfo字典的key。注意额外添加的属性都是readonly，NSNotification对象希望这些都是不可改变的

用类别来实现始终有一个问题，对于通知方法那个观察者是要知道的那个类别方法是要可见的（比如：audioPlayerDidStartPlackback:方法中没有定义hpe_endTime）。但是这些问题都可以在子类化NSNotification中得到解决。

##Typed parameters with subclassing

---

子类化NSNotification需要提供一个完整的观察者，包括参数，通知方法，自己的子类。我们的观察者协议变成如下样子：

{% highlight objective-c %}
@protocol HPEAudioPlayerObserver& lt;NSObject& gt;
@optional

-(void)audioPlayerDidStartPlayback:(HPEAudioPlayerDidStartPlaybackNotification*)notification;

-(void)audioPlayerDidFinishPlayback:(HPEAudioPlayerDidEndPlaybackNotification*)notification;

@end
{% endhighlight %}

在part1中NSNotification是一个类族，当调用init时抛出一个异常。这里实现NSNotification子类需要一些样板代码：

{% highlight objective-c %}
@interface HPEAudioPlayerNotification : NSNotification

-(id)initWithAudioPlayer:(HPEAudioPlayer*)audioPlayer;

@property (nonatomic, readonly) HPEAudioPlayer *audioPlayer;

@end

@interface HPEAudioPlayerDidStartPlaybackNotification : HPEAudioPlayerNotification

@end

@interface HPEAudioPlayerDidEndPlaybackNotification : HPEAudioPlayerNotification

-(id)initWithAudioPlayer:(HPEAudioPlayer*)audioPlayer endTime:(NSTimeInterval)endTime;

@property (nonatomic, readonly) NSTimeInterval endTime;

@end

@implementation HPEAudioPlayerNotification

-(id)initWithAudioPlayer:(HPEAudioPlayer *)audioPlayer {
    _audioPlayer = audioPlayer;
    return self;
}

@end

@implementation HPEAudioPlayerDidStartPlaybackNotification

-(NSString*)name {
    return @"HPEAudioPlayerDidStartPlaybackNotification";
}
@end

@implementation HPEAudioPlayerDidEndPlaybackNotification

-(id)initWithAudioPlayer:(HPEAudioPlayer*)audioPlayer endTime:(NSTimeInterval)endTime {
    if (self = [super initWithAudioPlayer:audioPlayer]) {
        _endTime = endTime;
    }
    return self;
}

-(NSString*)name {
    return @"HPEAudioPlayerDidEndPlaybackNotification";
}

@end
{% endhighlight %}

再次提醒属性都是readonly带参数的。


子类化方法需要提供一个完整的观察者，代价就是要创建一个类的每个方法处理子类化的NSNotification。如果不需要一个严格的类型，类别方法仍需要每个方法要正确的记录有效的参数。

##Further reading
--
- [Part 1: Receiving and testing notifications](http://iusn.github.io/NSNotificationCenter%20part%201/)
