---
layout: post
title: "我的玩具——乐高魔方机器人"
date: 2017-02-18 20:00:00
categories: lego nxt cube
comments: true
typora-root-url: ../../source
---

<iframe width="100%" height="450" src="https://www.youtube.com/embed/lXMsFn_69Dw" frameborder="0" allowfullscreen></iframe>

没梯子请点[优酷](http://player.youku.com/embed/XMjUyMDcxMDk1Mg==)

## 0x00 简要说明

**硬件环境**：Lego NXT 8547, iPhone 4S(需越狱)

**软件依赖**：LeJOS, BTStack, OpenCV2

**还原效率**：扫描10~15秒，计算<1秒，还原~1分钟

<!-- more -->

## 0x01 Lego & Tilted Twister

5年前心血来潮，为了做一个能自动拧魔方的机器人，买了一套乐高8547，按当时的收入算，简直是花了一笔巨款(默默心疼3秒)，之后就开始各种折腾。折腾了两天实在受不了那个中看不中用的GUI编程套件，直接刷了[LeJOS](http://www.lejos.org)(Lego Java OS)，它提供了Java Runtime，可以愉快地使用Java进行coding。

![lego-8547](/assets/2017/lego-8547.jpg)

先是抱Hans Andersson大神的大腿，开始拼[Tilted Twister](http://tiltedtwister.com/download/tt2/TT2.pdf)，拼成之后发现完全无法顺利行╮(╯▽╰)╭。主要原因是颜色识别很不准，主要是魔方的**橙色**块和**红色**块，通过颜色传感器采集到的颜色值太相近了(采集到的颜色值是8bit，范围是0~255)，根本无法有效区分开，外界光线的亮一点或者暗一点都会严重影响识别正确率。并且因为乐高自身的CPU和内存都弱到爆炸，大约需要30s~60s才能计算出一个平均50步的解法，再花约3-5分钟进行还原，速度还没有我自己快。

![lego-8547](/assets/2017/tiltedtwister2.jpg)

后来冒出了个想法：何不使用手机摄像头扫描，然后在手机上计算还原步骤再控制机器人还原？这样不仅可以一次性扫描9个色块，也可以利用手机强大的CPU在更短的时间内计算出步骤更少的解法，同时提高准确率与运行效率。

## 0x02 动手改造

先把颜色传感器拆掉，原来的LEGO的执行程序就不能用了，需要自己来写，只需实现机械控制部分。

机器人有两个Motor，以LEGO的前脸为正面视角：

![axis](/assets/2017/lego-axis.png)

- 机械臂往前`推(PUSH)`，可以让魔方绕X轴逆时针旋转90°，实现上、前、下、后四个面之间的翻转
- 底座`旋转(ROTATE)`，可以让魔方绕Y轴旋转，实现前，右，后，左四个面之间的翻转
- 机械臂和底座配合，可以将魔方翻转到任意面
- 机械臂`抓住(HOLD)`魔方的上面两层，然后底座旋转，可以实现`拧(TWIST)`魔方的底面
- 公式的每一步操作都可以拆解为：将该步骤所要拧的面翻转到底面，然后抓住魔方，用底座拧魔方的底面

(PS: 魔方公式形如`U2 F D' R2 D F2 B2 L2`,每个字母代表顺时针将一个面旋转90°：Up, Bottom, Front, Back, Left, Right. 字母后面跟`2`表示拧两次，也就是180°，跟`'`表示逆时针90°。)

因此，需要两个循环队列来保存魔方在X轴和Y轴方向上的状态：pushChain的第一个元素是朝下的面，第二个元素就是通过一次PUSH操作，会翻转到朝下的面，也就是朝前的面，以此类推。每次旋转或者翻转操作，都需要同步更新两个队列的状态

```java
    private static LinkedList<String>   pushChain = new LinkedList<String>(),
                                      rotateChain = new LinkedList<String>();

    static {
        pushChain.add(D);
        pushChain.add(F);
        pushChain.add(U);
        pushChain.add(B);
        rotateChain.add(F);
        rotateChain.add(L);
        rotateChain.add(B);
        rotateChain.add(R);
    }

    /**
     * push the cube using the long arm.
     * Face FRONT turns to DOWN
     */
    private static void push() {
        arm.rotate(PUSH_ANGLE);
        sleep(50);
        arm.rotate(-PUSH_ANGLE);
        // update pushChain and rotateChain
        pushChain.add(pushChain.remove(0));
        rotateChain.set(0, pushChain.get(1));
        rotateChain.set(2, pushChain.get(3));
    }

    /**
     * rotate the base (DOWN FACE) with a specified angle
     * @param angle the degrees to rotate
     * @param changeFacelet whether to update the facelet chain
     */
    private static void rotate(int angle, boolean changeFacelet) {
        int currentPosition = base.getTachoCount();
        switch (angle) {
            case 90:
                if (currentPosition > 225) {
                    base.rotateTo(currentPosition - 270);
                } else {
                    base.rotateTo(currentPosition + 90);
                }
                if (changeFacelet)
                    rotateChain.add(rotateChain.remove(0));
                break;
            case 180:
                if (currentPosition > 135) {
                    base.rotateTo(currentPosition - 180);
                }
                else {
                    base.rotateTo(currentPosition + 180);
                }
                if (changeFacelet) {
                    rotateChain.add(rotateChain.remove(0));
                    rotateChain.add(rotateChain.remove(0));
                }
                break;
            case -90:
                if (currentPosition < 45) {
                    base.rotateTo(currentPosition + 270);
                } else {
                    base.rotateTo(currentPosition - 90);
                }
                if (changeFacelet)
                    rotateChain.add(0, rotateChain.remove(3));
                break;
            default:
        }
        if (changeFacelet) {
            pushChain.set(1, rotateChain.get(0));
            pushChain.set(3, rotateChain.get(2));
        }
    }
```

之后就可以将任意面，通过不超过两步的操作翻转到底面，例如，可以通过两次`PUSH`将顶面翻转到底面，也可以通过底座顺时针旋转90°后，再`PUSH`将左侧的面翻转到底面：

```Java
    /**
     * make the specified facelet downwards to bottom
     * @param facelet target face
     */
    public static void changetoFacelet(String facelet) {
        switch (pushChain.indexOf(facelet)) {
            case 0:
                return;
            case 1:
                push();
                return;
            case 2:
                push();
                push();
                return;
            case 3:
                rotate(180, true);
                push();
                return;
            default:
        }
        switch (rotateChain.indexOf(facelet)) {
            case 1:
                rotate(90, true);
                push();
                return;
            case 3:
                rotate(-90, true);
                push();
                return;
            default:
        }
    }
```

`拧(TWIST)`的动作可以通过`HOLD->ROTATE->RELEASE`的步骤实现

```java
    /**
     * twist DOWN FACE 90 degrees
     */
    private static void turnClockwise() {
        hold();
        rotate(90, false);
        release();
    }

    /**
     * twist DOWN FACE -90 degrees
     */
    private static void turnAntiClockwise() {
        hold();
        rotate(-90, false);
        release();
    }

    /**
     * twist DOWN FACE 180 degrees
     */
    private static void turnSemiCycle() {
        hold();
        rotate(180, false);
        release();
    }
```

至此，输入公式，机器人就可以一步步操作了

```java
     /**
     * Execute a specified expression
     * @param exp the expression to be executed
     */
    static void execute(String exp) {
        for (int i = 0; i < exp.length(); i++) {
            changetoFacelet(exp.substring(i, i + 1));
            if (i == exp.length() - 1 || exp.charAt(i + 1) == ' ') {
                turnClockwise();
                i++;
            } else if (exp.charAt(i + 1) == '\'') {
                turnAntiClockwise();
                i += 2;
            } else if (exp.charAt(i + 1) == '2') {
                turnSemiCycle();
                i += 2;
            }
        }
    }
```

下面的视频是当时留下的一个DEMO，事先向LEGO输入了20步的还原公式，LEGO只管按照公式拧：

<iframe height=490 width='100%' src='http://player.youku.com/embed/XMzc0MTgzNDEy' frameborder=0 'allowfullscreen'></iframe>

## 0x03 加入Android控制器

LEGO支持蓝牙通信，因此可以用手机做主控端，整个系统的构思如下：

- 手机与LEGO通过蓝牙连接
- LEGO检测到魔方放入之后通知手机开始扫描
- 手机扫描完一个面之后，通知LEGO将魔方翻转到下一个面
- 扫描完毕后，手机开始计算还原步骤
- 手机通过蓝牙将还原公式发送给LEGO
- LEGO按照公式将魔方还原

手里有个Android手机(KTouch-650)，还有一部iPod Touch 4，虽然Android机性能有点差，但我那时候完全不懂Android和iOS开发。好在Java技能是游刃有余的，Android开发可以快速上手，也就不得不选择Android了。蓝牙连接和还原算法都好办，虽然LeJOS官方没有提供Android与LEGO通信的SDK，但是完全可以仿照PC的SDK实现一套，将蓝牙相关的实现替换为android.bluetooth包提供的实现即可。网上已经有大神给出了[源码](https://github.com/jpralves/tourrobot/blob/master/NXTController/src/lejos/pc/comm/NXTCommAndroid.java)。还原算法可以直接采用Java实现的Two-Phase算法[twophase.jar](http://kociemba.org/cube.htm)。

那么问题来了，怎么检测魔方在摄像头采集的画面中的位置，或者说怎么确定采集哪些像素点的颜色？最笨的办法就是——固定位置，为此，在取景界面上绘制了一个9宫格作为参考线，方便手动对齐：

```java
@Override
public void onDraw(Canvas canvas) {
	int width = this.getWidth();
	int height = this.getHeight();
	Paint paint = new Paint();
	paint.setColor(Color.WHITE);
	paint.setStyle(Style.STROKE);
	canvas.drawRect(0,0,width-1,height-1,paint);
	for(int i = 1; i < 3; i++) {
		canvas.drawLine(width/3*i,0,width/3*i,height-1,paint);
		canvas.drawLine(0, height/3*i, width-1, height/3*i, paint);
	}
}
```

大概长成这个样子：

![grid](/assets/2017/grid.png)

手动将魔方与九宫格参考线对齐之后，点击屏幕任意位置开始取色。取色的时候需要取中心区域的多个点，然后计算平均色值，避免单个点的色值误差太大。这里遇到一个问题是Android摄像头的预览图像数据是YUV色彩空间，而不是RGB，需要先进行一次转换:

```java
@Override
public void onPreviewFrame(byte[] data, Camera camera) {
	if(!flag)return;
	flag = false;
	Parameters parameters = camera.getParameters();
	int width = parameters.getPreviewSize().width;
	int height = parameters.getPreviewSize().height;
	int[] rgbBuf = new int[height*width];
	decodeYUV420SP(rgbBuf, data, width, height);
	Bitmap bitmap = Bitmap.createBitmap(rgbBuf, width, height, Config.RGB_565);
	pickupColors(bitmap);
	if(!debug && faceletNum != 6)
	try {
		out.writeInt(1);
		out.flush();
	} catch (IOException e) {
	}
}

private void decodeYUV420SP(int[] rgbBuf, byte[] yuv420sp, int width,
		int height) {
	final int frameSize = width * height;
	if (rgbBuf == null)
		throw new NullPointerException("buffer 'rgbBuf' is null");
	if (rgbBuf.length < frameSize)
		throw new IllegalArgumentException("buffer 'rgbBuf' size "
				+ rgbBuf.length + " < minimum " + frameSize);
	if (yuv420sp == null)
		throw new NullPointerException("buffer 'yuv420sp' is null");
	if (yuv420sp.length < frameSize * 3 / 2)
		throw new IllegalArgumentException("buffer 'yuv420sp' size "
				+ yuv420sp.length + " < minimum " + frameSize * 3 / 2);
	for (int j = 0, yp = 0; j < height; j++) {
		int uvp = frameSize + (j >> 1) * width, u = 0, v = 0;
		for (int i = 0; i < width; i++, yp++) {
			int y = (0xff & ((int) yuv420sp[yp])) - 16;
			if (y < 0)
				y = 0;
			if ((i & 1) == 0) {
				v = (0xff & yuv420sp[uvp++]) - 128;
				u = (0xff & yuv420sp[uvp++]) - 128;
			}
			int y1192 = 1192 * y;
			int r = (y1192 + 1634 * v);
			int g = (y1192 - 833 * v - 400 * u);
			int b = (y1192 + 2066 * u);
			if (r < 0)
				r = 0;
			else if (r > 262143)
				r = 262143;
			if (g < 0)
				g = 0;
			else if (g > 262143)
				g = 262143;
			if (b < 0)
				b = 0;
			else if (b > 262143)
				b = 262143;
			rgbBuf[yp] = 0xff000000 | ((r << 6) & 0xff0000)
					| ((g >> 2) & 0xff00) | ((b >> 10) & 0xff);
		}
	}
}
```

转换到RGB就可以正常取色，取色完毕之后就需要根据颜色来计算魔方的状态了，然后转换为U、B、D、F、R、L的形式表示。我采取的算法比较简单，先确定每个面中心块的颜色，做为该面的颜色(因为魔方不管怎么转，一个面的中心块，是不会跑到其他面的)，然后拿每个块的颜色分别与六个中间块的颜色对比，计算在RGB分量上的差值，差值最小的中心块的颜色，就是当前色块的颜色，也就是说这个块还原后和这个中心块在同一个面。差值计算方式采用RGB分量的差平方之和，代码节选如下：

```java
private void solveColors() {
	int centerColors[] = new int[6];
	char[] facelets = {'U','B','D','F','R','L'};
	for(int i=0; i<6; i++) {
		centerColors[i] = cubeColors[i*9+4];
		int color = centerColors[i];
		Log.d("color","center color: " + facelets[i] + " " + Color.red(color)+"/"+Color.green(color)+"/"+Color.blue(color));			
	}
	for(int i=0; i<54; i++) {
		int min = Integer.MAX_VALUE;
		int cubeR = Color.red(cubeColors[i]),
			cubeG = Color.green(cubeColors[i]),
			cubeB = Color.blue(cubeColors[i]);
		Log.d("color","cube color: " + cubeR+"/"+cubeG+"/"+cubeB);			
		for(int j=0; j<6; j++) {
			int centerR = Color.red(centerColors[j]),
				centerG = Color.green(centerColors[j]),
				centerB = Color.blue(centerColors[j]);
			int s = (cubeR - centerR) * (cubeR - centerR)
					+ (cubeG - centerG) * (cubeG - centerG)
					+ (cubeB - centerB) * (cubeB - centerB);
			Log.d("color","s for " + facelets[j] + ":" + s);
			if(s < min) {
				min = s;
				cubefaces[i] = facelets[j];
			}
		}
		Log.d("color","face: "+cubefaces[i] + " " + i);
	}
}
```

然而，实际测试下来发现，这样的算法经常会有计算错误的时候，光线比较暗的情况下，摄像头取到的24bit的橙色和红色，依然很接近，用肉眼都很难区分。

还有另外一个坑，采用twophase.jar这个lib，需要约30~60M的内存，用于构建搜索树，这在当时的Android机上(至少在我的破手机上)，是不可能的事情，只好作罢，方案不得不改成：

1. Android连接LEGO并手动扫描魔方状态
2. 电脑上起一个Servlet，用于跑还原算法，按照i5的性能，平均1~2秒即可计算出22步以内的结果。
3. Android通过HTTP调用电脑的计算接口，把魔方状态作为参数，获取还原公式
4. 将还原公式发给LEGO，由LEGO还原魔方

![woqu](/assets/2017/woqu.png)

如此繁琐，颜色识别的准确率又不美丽，我不禁开始思考人生。。。

![thinking](/assets/2017/thinking.png)

期间考虑过换用iOS设备，但是iOS设备的蓝牙不支持RFCOMM通讯协议，也就作罢。

结果是，机器人被供了起来，5年里，跟着我搬了4次家。。。

## 0x04 在iOS上重构控制器

后来慢慢接触到了iOS越狱开发，也知道了BTStack这个开源库，可以通过直接操作底层接口，让越狱的iOS设备实现RFCOMM等官方蓝牙SDK不支持的协议。看了一眼躺在书橱里吃灰的LEGO机器人，想着是时候拿出来晒晒太阳了 ^.^

### 蓝牙部分

先clone了BTStack的[源码](https://github.com/bluekitchen/btstack)，编译的时候遇到很多错误，iOS部分的工程结构本身就有很多问题，还有很多符号找不到。后来chekcout了v0.9分支，发现master的工程结构跟v0.9的完全不一样，但是两个分支里iOS的文件夹别无二致，缺少的符号在v0.9分支里都有，可能是长时间没有维护iOS版本的库了把，最终使用v0.9分支成功编译。其实也可以直接在Cidya里安装BTStack，然后将libBTStack.dylib从手机里拷出来。

接下来要实现RFCOMM通讯功能，坑爹的是，BTStack给出的demo中使用的是L2CAP协议，同时，BTstackManager类中只有对底层协议的封装，没有对RFCOMM/L2CAP等高层数据传输协议进行封装，留了接口，但是方法体是空的，只好自己动手了：

```objective-c
-(void) handlePacketWithType:(uint8_t)packet_type forChannel:(uint16_t)channel andData:(uint8_t *)packet withLen:(uint16_t) size {
    switch (state) {
        // -- omitted --
        case kActivated:
            switch (packet_type) {
                case HCI_EVENT_PACKET:
                    switch (packet[0]){
                        case BTSTACK_EVENT_STATE:
                            [self activationHandleEvent:packet withLen:size];
                            break;
                            
                        case RFCOMM_EVENT_OPEN_CHANNEL_COMPLETE:
                            if (packet[2]) {
                                printf("RFCOMM channel open failed, status %u\n", packet[2]);
                                // TODO connection failed callback
                            } else {
                                uint16_t rfcomm_channel_id = READ_BT_16(packet, 12);
                                uint16_t mtu = READ_BT_16(packet, 14);
                                printf("RFCOMM channel open succeeded. New RFCOMM Channel ID %u, max frame size %u\n", rfcomm_channel_id, mtu);
                                [self.rfcommDelegate rfcommConnectionCreatedAtAddress:"" forChannel:channel asID:rfcomm_channel_id];
                            }
                            break;
                            
                        case HCI_EVENT_DISCONNECTION_COMPLETE:
                            printf("Basebank connection closed\n");
                            uint16_t rfcomm_channel_id = READ_BT_16(packet, 12);
                            [self.rfcommDelegate rfcommConnectionClosedForConnectionID:rfcomm_channel_id];
                            break;
                        default:
                            break;
                    }
                    break;
                case RFCOMM_DATA_PACKET:
                    NSLog(@"Received from 0x%02X %@", channel, [NSData dataWithBytes:packet length:size]);
                    [self.rfcommDelegate rfcommDataReceivedForConnectionID:channel withData:packet ofLen:size];
                    break;
            }
            [self discoveryHandleEvent:packet withLen:size];
            break;
        default:
            break;
    }
    // -- omitted --
}
-(BTstackError) createRFCOMMConnectionAtAddress:(bd_addr_t*) address withChannel:(uint16_t)channel authenticated:(BOOL)authentication {
    if (state < kActivated) return BTSTACK_NOT_ACTIVATED;
    if (state != kActivated) return BTSTACK_BUSY; 
    bt_send_cmd(&rfcomm_create_channel, address, channel);
    return 0;
};
-(BTstackError) sendRFCOMMPacket:(NSData*)packet ForConnectionId:(uint16_t)connectionId {
    if (state < kActivated) return BTSTACK_NOT_ACTIVATED;
    NSLog(@"Send to 0x%02X %@", connectionId, packet);
    bt_send_rfcomm(connectionId, (uint8_t *)[packet bytes], [packet length]);
    return 0;
};
-(BTstackError) closeRFCOMMConnectionWithID:(uint16_t) connectionID {
    if (state <kActivated) return BTSTACK_NOT_ACTIVATED;
    bt_send_cmd(&hci_disconnect,connectionID);
    return 0;
};
```

RFCOMM部分的代码实现完毕，尝试发送纯文本数据，结果LEGO根本毫无反应o(╯□╰)o。仔细阅读了LeJOS蓝牙部分的源码，发现数据包的头部，被添加了两个字节(Big-endian)用于标识数据包的大小:

```java
public void write(byte[] data) throws IOException {
    // Send length of packet (Least and Most significant byte)
    // * NOTE: Bluetooth only. 
    os.write((byte)data.length);
    os.write((byte)(data.length >>> 8));
    os.write(data);
    os.flush();
}
```

这下就好办了，发送数据之前，先处理一下数据包，同样在头部插入两个字节的长度即可:

```objective-c
// Util.m
+(NSData *) btDataForNxtWithString: (NSString*)string {
    const void* bytes = string.UTF8String;
    NSInteger length = string.length;
    u_int8_t* data = malloc(length + 2);
    data[0] = length & 0xff;
    data[1] = (length >> 8) & 0xff;
    memcpy(data + 2, bytes, length);
    NSData* _data = [NSData dataWithBytes:data length:length + 2];
    free(data);
    NSLog(@"Send Data: %@", _data);
    return _data;
}
```

深夜3点，测试一下RFCOMM通信

![bluetooth](/assets/2017/bluetooth.jpg)

真是一把辛酸泪。。。

![cry](/assets/2017/cry.png)

### 扫描部分

之前Android控制器的扫描部分存在两个遗留问题，一是无法自动采集魔方的颜色，二是对颜色的识别仍然存在不准的情况。这两个问题不解决，始终是心里的一个疙瘩。

如何自动采集魔方的颜色呢？如果能检测到魔方的位置，一切就好办了。那么问题转换成如何检测摄像头的画面中是否有魔方，以及魔方在画面中的位置。我先考虑了一种方案：

- 检测画面中的直线，如果有其中8条直线能够组成一个九宫格的样式，则检测到魔方

怎么检测直线呢？经过一番Google，找到了基于iOS开源图像处理框架GPUImage的一种算法。事实证明，我还是too young, too naive：

![detect1](/assets/2017/detect1.png)

不管如何调整检测参数，直线的数量都超乎我的想象，更何况，我根本想不出一种算法去判断他们是不是能组成九宫格的图案。。。

后来就到处扒图像识别相关的资料，发现了基于OpenCV的一个很有意思的[Demo](https://github.com/opencv/opencv/blob/master/samples/cpp/squares.cpp)，可以识别图像中的正方形，新的识别方案就此诞生

- 如果能从图像中检测到9个相互不覆盖的正方形，就可以近似认为检测到了魔方

![detect2](/assets/2017/detect2.png)

```objective-c
void findSquares( Mat& image, vector<vector<cv::Point> >& squares )
{
    squares.clear();
    vector<vector<cv::Point> > contours;
    // find contours and store them all as a list
    findContours(image, contours, CV_RETR_LIST, CV_CHAIN_APPROX_SIMPLE);

    vector<cv::Point> approx;

    // test each contour
    for( size_t i = 0; i < contours.size(); i++ )
    {
        // approximate contour with accuracy proportional
        // to the contour perimeter
        approxPolyDP(Mat(contours[i]), approx, arcLength(Mat(contours[i]), true)*0.05, true);

        // square contours should have 4 vertices after approximation
        // relatively large area (to filter out noisy contours)
        // and be convex.
        // Note: absolute value of an area is used because
        // area may be positive or negative - in accordance with the
        // contour orientation
        if( approx.size() == 4 &&
           fabs(contourArea(Mat(approx))) > 1000 &&
           isContourConvex(Mat(approx)) )
        {
            double maxCosine = 0;

            for( int j = 2; j < 5; j++ ){
                // find the maximum cosine of the angle between joint edges
                double cosine = fabs(angle(approx[j%4], approx[j-2], approx[j-1]));
                maxCosine = MAX(maxCosine, cosine);
            }

            double lineLength1 = distance(approx[0], approx[1]);
            double lineLength2 = distance(approx[1], approx[2]);


            // if cosines of all angles are small and border length almost equals
            // (all angles are ~90 degree) then write quandrange
            // vertices to resultant sequence
            // then filter out big squares such as the whole facelet of the cube
            if( maxCosine < 0.1
               && fabs(lineLength1 - lineLength2) / MAX(lineLength1, lineLength2) < 0.1
               && lineLength1 / image.cols < 0.3) {

                if(squares.empty()) {
                    squares.push_back(approx);
                } else {
                    // make sure no overlap
                    for(vector<vector<cv::Point>>::iterator s = squares.begin(); s < squares.end(); s++) {
                        int contains = squareContains(*s, approx);
                        if (contains == 1){ //s contains approx
                            squares.erase(s);
                            squares.push_back(approx);
                            break;
                        } else if(contains == -1) { // approx contains s
                            break; // discard this approx
                        } else {
                            if (s == squares.end() - 1) {
                                squares.push_back(approx);
                                break;
                            }
                        }
                    }
                }
            }
        }
    }
    // printf("found %lu squares\n", squares.size());

    if(squares.size() != 9) {
        squares.clear();
    } else {
        // sort squares to sequence
        // 0 1 2
        // 3 4 5
        // 6 7 8
        sort(squares.begin(), squares.end(), compareTwoPointsWithY);
        sort(squares.begin()+0, squares.begin()+3, compareTwoPointsWithX);
        sort(squares.begin()+3, squares.begin()+6, compareTwoPointsWithX);
        sort(squares.begin()+6, squares.begin()+9, compareTwoPointsWithX);
    }
}
```

是不是很机智！当然，这个算法也会有傻逼的时候：

![detect3](/assets/2017/detect3.png)

解决类似的问题需要再检测一下每个正方形之间的间距。不过，如果放在魔方机器人的底座上进行扫描，周围应该没有什么干扰项，多增加一部分计算的代码反而会影响画面的刷新率，也就无所谓了。

扫描完魔方的54个块之后，就需要对每个块的颜色进行识别分组，之前的算法是计算每个颜色与6个基准色(也就是每个面中心块的颜色)的色差，仍然会存在不准的情况，这次我扫描记录了大量颜色数据，从中分析出以下特征：

- 白色块的RGB分量之和大于任何其他色块
- 绿色块的G分量与R/B分量的差值，是所有色块中最大的
- 除去白色与绿色，剩下的色块，B分量与R/G分量的差值，从大到小依次是
  - 蓝色 > 红色 > 橙色 > 黄色

其实这样的比较方式，在昏暗的光线下，红色与橙色仍然非常相近，但根据测试，错误率大概不到3%，还算可以接受。因此，只需要对54个颜色进行多次不同维度的排序，就可以识别出正确的颜色

```objective-c
- (NSString *) detectColors {
    sort(colorNodes.begin(), colorNodes.end(), sortForWhite); // W X X X X X
    sort(colorNodes.begin() + 9 , colorNodes.end(), sortForGreen); // W G X X X X
    sort(colorNodes.begin() + 18 , colorNodes.end(), sortForBlue); // W G B R O Y
    char colors[] = "WWWWWWWWWGGGGGGGGGBBBBBBBBBRRRRRRRRROOOOOOOOOYYYYYYYYY";
    for(int i = 0; i < colorNodes.size(); i ++) {
        colorNodes[i].color = colors[i];
        
    }
    sort(colorNodes.begin(), colorNodes.end(), restoreInputSequence);
    map<char, char> colorMap;
    colorMap[colorNodes[4].color] = 'U';
    colorMap[colorNodes[13].color] = 'B';
    colorMap[colorNodes[22].color] = 'D';
    colorMap[colorNodes[31].color] = 'F';
    colorMap[colorNodes[40].color] = 'R';
    colorMap[colorNodes[49].color] = 'L';
    
    int convertTable[] = {
        8,7,6,5,4,3,2,1,0,
        38,41,44,37,40,43,36,39,42,
        35,34,33,32,31,30,29,28,27,
        26,25,24,23,22,21,20,19,18,
        47,50,53,46,49,52,45,48,51,
        9,10,11,12,13,14,15,16,17
    };
    
    char state[55];
    state[54] = 0;
    printf("============================\n");
    for(int i = 0; i < colorNodes.size(); i ++) {
        state[i] = colorMap[colorNodes[convertTable[i]].color];
        printf("color %d: %c %f, %f, %f\n", i, colorNodes[i].color, colorNodes[i].scalar.val[2], colorNodes[i].scalar.val[1], colorNodes[i].scalar.val[0]);
    }
    return [[NSString alloc]initWithCString:state encoding: NSUTF8StringEncoding];
}
```

PS: `convertTable`是为了将输出的颜色顺序转换为还原算法所需的顺序。

PPS: 输出的颜色是用该颜色所属的面表示的。

### 算法部分

Two-Phase算法，也有pure C的版本，但是这个算法内存占用奇高，且计算出的还原步骤一般都要22步，甚至更多，如果要计算20步以内([上帝之数是20](http://www.cube20.org/)，也就是说任意魔方都可以用不超过20步进行还原)的解法，要花上数分钟的时间。因此我又花了大量的时间寻找性能更好的算法，最终找到两个：

- optimal Rubik's cube solver，需要80M内存，但实际测试，有大约50%的情况，一直运行，但始终给不出答案(超过5分钟)，不知道是不是算法中存在bug
- Dik T. Winter 的算法，内存占用大概只有10M+，计算任意魔方的解法，几乎都可以瞬间给出20步以内的结果

很显然，这个没有名字的算法正合我意。经过一番调整和优化，这个算法顺利地在iOS上跑了起来。

将这三部分整合起来，就是文章最开始的那个视频的样子，历经千辛万苦，终于实现了最初设计的那套方案。

![xuebi](/assets/2017/xuebi.jpg)

## 0x05 膜拜一下大神们

1. `3.25s`的世界纪录保持者Cube Stormer 3

   <iframe width="100%" height="450" src="https://www.youtube.com/embed/X0pFZG7j5cE" frameborder="0" allowfullscreen></iframe>

   这个是使用Lego NXT的升级版 EV3 拼成的魔方机器人，猜猜用了多少零件？

2. 国内LEGO大神，动力老男孩做的[萝卜头](http://www.diy-robots.com/?page_id=46)

3. 开源魔方机器人，[MindCuber](http://mindcuber.com/index.html)，作者正是Cube Stormer 3的作者之一`David Gilday`