# weixin-video


微信的video问题android下全屏和 ios下不自动播放是个比较顽固的问题。下面介绍接种解决方案。

###更新，发现一种新的方式

	  <video id="videoID" src="xxx" x-webkit-airplay="allow" preload="meta" webkit-playsinline="true" x5-video-player-type="h5-page" x5-video-player-fullscreen="true" playsinline="true" poster="">



### 【原生video hask】（主要是解决android全屏问题）

#### 第一种： H5类型全屏播放

这种方式基本是现在的主流，如果有交互可以将交互操作的对象悬浮在播放器上方。缺点就是会拉伸一下屏幕全屏，比较恶心

	<video id="videoID" src="XXX" poster="XXX.jpg" preload="auto" x5-video-player-type='h5' x5-video-player-fullscreen='true' x-webkit-airplay="true"  webkit-playsinline="true" playsinline="true"></video>

主要标志就是x5-video-player-type='h5'，也有人统一封了下用canvas来绘制。


#### 第二种：非H5播放器，不全屏播放

微信的video标签有一个*x5-playsinline*属性，与ios的webkit-playsinline功效是一样的，且在android下生效。 但是还是有一些不同：

	 <video id="J_video"  x-webkit-airplay="allow"  x5-playsinline="true" webkit-playsinline="true" playsinline="true" poster="" preload="auto" src="./m2.mp4">

需要注意的是不能与*x5-video-player-type='h5'*和*x5-video-player-fullscreen='true'*来共同使用，这样的结果导致视频其实是脱离了文档流的，而且层级是最高的，不能覆盖.想要一些交互效果就别想了。并且这种方式无法使用canvas绘制。

#### 第三种：JS实现动画帧

这种方式只适用于比较短的视频，比如几秒。可以用ffmpeg导出帧图，然后用各种js引擎实现动画。


> 关于无法自动退出全屏或者播放器可以播放一个音频的方式来解决。

这几种微信android的解决方案总是那么不完美，各有各的问题。下面介绍两种纯js的基本完美解决方案

### 【[JSmpeg](https://github.com/phoboslab/jsmpeg)】

支持MPEG1 Video 和 MP2 Audio ，推荐以25fps的帧率解码 720p的视频。播放时视频是逐步分段加载的。事件API交匮乏。

> 视频处理 `可以通过降低码率的方式减小视频大小`，有两种方式：

> 1 静态文件

> 2 stream流，可以通过node来实现

**静态文件方式**

首先先转下码

	ffmpeg -i in.mp4 -f mpegts \
	-codec:v mpeg1video -s 960x540 -b:v 1500k -r 25 -bf 0 \
	-codec:a mp2 -ar 44100 -ac 1 -b:a 128k \
	out.ts


前端代码超简单：

	var player = new JSMpeg.Player('out.ts',{
			    loop: false,
			    autoplay: false,
			    canvas:document.querySelector('canvas'),
			    chunkSize:1024*512   //每段加载的视频块大小 此处512k
			});

	function play(){
		player.play();
	}
	
	//player.stop();
	//player.pause();
	//player.currentTime
	//player.isPlaying
	//player.volume = 1

> ios下你会发现没有声音，很多人说不支持ios. 实际是因为ios的安全机制，必须用户点击才能播放音视频，这里视频虽然绕路运行了，但是音频调用的是音频api，依然受ios限制，解决办法有两种：

			#1 点击事件中解锁音频
			document.addEventListener('touchstart',function(){
				player.audioOut.unlock();
				this.removeEventListener(e.type,arguments.callee,false);
			},false);
			
			#2 微信hack，需要引入微信 js, 然后是hack代码：
			if ( navigator.userAgent.indexOf( 'MicroMessenger' ) !== -1 ) {
		        document.addEventListener("WeixinJSBridgeReady", function () {
		           	player.audioOut.unlock();
		        });
		    }
			
			

**node服务器stream方式**

这个自行查看git上的说明来弄吧



### 【[ogv.js](https://github.com/brion/ogv.js)】

> 一个支持 Ogg/Vorbis/Theora/Opus/WebM格式的基于Emscripten编译的JavaScript播放器。播放器的操作和API类似Video标签。

> 转换ofv,ogg,webp啥的需要用到*libtheora和libvorbis和webp*, 建议重新编译安装ffmpeg（完整些，以防后患）

	brew reinstall ffmpeg --with-fdk-aac --with-fontconfig --with-freetype --with-frei0r --with-game-music-emu --with-libass --with-libbluray --with-libbs2b --with-libcaca --with-libgsm --with-libmodplug --with-librsvg --with-libsoxr --with-libssh --with-libvidstab --with-libvorbis --with-libvpx --with-opencore-amr --with-openh264 --with-openjpeg --with-openssl --with-opus --with-rtmpdump --with-rubberband --with-sdl2 --with-snappy --with-speex --with-srt --with-theora --with-tools --with-two-lame --with-wavpack --with-webp --with-x265 --with-xz --with-zeromq --with-zimg

> 视频处理

	#ogv
	ffmpeg -i source.mp4 -codec:v libtheora -qscale:v 7 -codec:a libvorbis -qscale:a 5 output.ogv
	
	#webp
	ffmpeg -y -i source.mp4 -vcodec libvpx -quality good -cpu-used 5 -b:v 700k -maxrate 700k -bufsize 1000k -qmin 10 -qmax 42 -vf  -threads 4 -acodec libvorbis -f webm output.webm


> 代码

	<script src="ogv.js"></script>

	var player = new OGVPlayer();
	document.body.appendChild(player);

	// player.src = '../curiosity.ogv';
	player.src = '../m2.webm';

	player.play();

	player.addEventListener('ended', function() {
		console.log('ended');
	});


> 同样播放收到ios安全机制限制，解决办法基本和fsmpeg相仿
	
	#1 点击事件中解锁音频
	document.addEventListener('touchstart',function(){
		player.play();
		player.stop();
		this.removeEventListener(e.type,arguments.callee,false);
	},false);
	
	#2 微信hack，需要引入微信 js, 然后是hack代码：
	if ( navigator.userAgent.indexOf( 'MicroMessenger' ) !== -1 ) {
        document.addEventListener("WeixinJSBridgeReady", function () {
           	player.play();
           	player.stop();
        });
    }
