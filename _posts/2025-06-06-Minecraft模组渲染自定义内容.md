---
categories: [开发]
tags: [Minecraft]
---

## 渲染文字

最简单的渲染方式：
```java
GuiGraphics graphics = ...;
graphics.drawString(font, component, x, y, argbColor, shouldUseDropShadow);
```
其中`shouldUseDropShadow`表示了是否要渲染下沉阴影，`argbColor`决定了没有定义颜色的文字部分使用的默认颜色。

但是这样的渲染在和其它内容同时渲染且产生覆盖时存在渲染结果不符合预期的问题，笔者对该问题出现的原因无法理解，现对该问题的情况描述如下： 
1. graphics.drawString默认使用Font.DisplayMode.Normal，从结果来看总是会显示在最前
2. 这很奇怪，因为三种字体渲染模式都没有写入深度信息，而SEE_THROUGH是唯一的NO_DEPTH_TEST，另外两个都是LEQUAL_DEPTH_TEST 
3. 如果以绘制顺序看的话，先绘制文字那文字就应该可以被覆盖，但结果是文字始终在最前 
4. 如果假设drawInBatch始终是最后绘制的话，SEE_THROUGH的无深度测试更应该显示在最前了，但是反而是可以正常覆盖了 
5. 从结果来看，SEE_THROUGH可以实现正确的覆盖顺序

正确的能满足覆盖顺序要求的渲染代码如下：
```java
GuiGraphics graphics = ...;
PoseStack pose = graphics.pose();
pose.pushPose();
// x, y, scale都为float类型
if (scale != 1) {
    pose.scale(scale, scale, scale);
    // 如果有缩放，应该对坐标进行反向缩放，以确保左上顶点的坐标仍符合预期
    x = x / scale;
    y = y / scale,
}
// 该方法传入的参数主要参考graphics.drawString内部使用的参数
font.drawInBatch(
    textComponent,
    x, y,
    argbColor, shouldUseDropShadow,
    pose.last().pose(),
    graphics.bufferSource(), Font.DisplayMode.SEE_THROUGH,
    0, 0xF000F0
);
pose.popPose();
```


## 渲染静态图片

对普通图片可以使用自带的DynamicTexture完成，通过如下方式注册材质：
```java
// texturePath不需要与真实文件挂钩，自己定义一个人类可读的不重复path即可
var loc = ResourceLocation.tryBuild(MOD_ID, texturePath);
// 需要准备一份图片数据，图片需为png格式
var texture = new DynamicTexture(NativeImage.read(pngData));
Minecraft.getInstance().getTextureManager().register(loc, texture);
```
> `NativeImage.read(byte[])`对较大图片的处理有问题，请尽量使用`NativeImage.read(InputString)`  
{: .prompt-warning }

> 材质注册一次即可，不再需要该材质后应及时调用textureManager.release(loc)释放资源  
{: .prompt-warning }

材质注册后在需要渲染的时候通过如下方式渲染：
```java
// 各参数含义如下：
// x, y, width, height: 渲染坐标和宽高
// uOffset, vOffset, uWidth, vHeight: 要渲染的图片内容在之前注册的材质中的坐标和宽高
// textureWidth, textureHeight: 之前注册的材质的宽高
// 如果要渲染整张图片，则uOffset = 0, vOffset = 0, uWidth = textureWidth, vHeight = textureHeight
graphics.blit(loc, x, y, width, height, uOffset, vOffset, uWidth, vHeight, textureWidth, textureHeight);
```

### 渲染动态图片(通过静态材质绘制部分区域的方式，推荐使用)
在静态图片渲染的基础上，可以通过简单的范围渲染的方式实现动态图片。  
流程如下：
1. 把所有帧绘制到一张图片上
2. 每次渲染前检查当前处于哪一帧，并得到当前帧在整张图片上的坐标偏移
3. 根据坐标偏移绘制图片，参数参考上面的`graphics.blit`

相关参数的伪代码参考如下(实际渲染时请注意尽量减少渲染期间的计算量)：
```java
var frameCount = ...;
var frameWidth = ...;
var frameHeight = ...;
var frameTiming = ...;
var rowCount = (int) Math.sqrt(frameCount);
var textureWidth = width * rowCount;
var textureHeight = height * Math.ceilDiv(frameCount, rowCount);

var frameIdx = ...;
var currentFrameStart = ...;
checkFrame() {
  if (System.currentTimeMillis() - currentFrameStart > frameTiming[frameIdx]) {
    frameIdx = (frameIdx + 1) % frameTiming.length;
    currentFrameStart = System.currentTimeMillis();
    uOffset = (frameIdx % rowCount) * frameWidth;
    vOffset = (frameIdx / rowCount) * frameHeight;
  }
}
graphics.blit(resLoc,
    x, y, width, height,
    uOffset, vOffset, frameWidth, frameHeight,
    textureWidth, textureHeight
);
```

### 渲染动态图片(通过动态材质的方式)
> 动态材质的方式由于是通过tick进行帧更新的，最低精度显然是1tick=50ms，如果需要更高精度的帧持续时间，应该使用静态材质绘制部分区域的方式  
{: .prompt-tip }

如果要通过动态材质的方式渲染动态图片，需要新建一个类继承AbstractTexture并实现Tickable，然后在tick函数中处理更新。  
一个简单的方式是对每个不同的图片都创建一个NativeImage，并记录每个图片显示的tick时长。相关逻辑如下：

在构造函数中：
```java
// this.getId()是父类中的方法
if (!RenderSystem.isOnRenderThread()) {
    RenderSystem.recordRenderCall(() -> {
        TextureUtil.prepareImage(this.getId(), frameWidth, frameHeight);
    });
} else {
    TextureUtil.prepareImage(this.getId(), frameWidth, frameHeight);
}
```

在`tick`函数中进行如下的实现(参考了Minecraft原版动态材质的逻辑)：
```java
@Override
public void tick() {
    // tick函数每tick都会被调用一次(正常20tick/s)
    if (!RenderSystem.isOnRenderThread()) {
        RenderSystem.recordRenderCall(this::cycleAnimationFrames);
    } else {
        this.cycleAnimationFrames();
    }
}

private void cycleAnimationFrames() {
    // 自行处理图片切换逻辑
    // 每次切换图片都应该调用下面的方法来更新材质
    this.bind(); // 这个是父类中的方法，每次更新时调用一次来绑定当前要修改的纹理
    nativeImage.upload(0, 0, 0, 0, 0, frameWidth, frameHeight, false, false);
    // 注意：upload函数最后一个参数必须是false，否则会自动close导致无法进行下一次更新。
}
```
> `NativeImage.upload`的参数说明如下：  
第1个参数是midMapLevel，一般不会为动图做midMap，所以一般为0，同时第8个参数midMap也为false  
第2、3个参数用于纹理偏移(可以理解为数组复制的dstOffset)，通常为0即可(除非有只更新部分纹理的需求)  
第4、5个参数用于图片偏移(可以理解为数组复制的srcOffset)，根据对应帧在完整图片中的位置提供  
第6、7个参数为要更新的矩形区域的宽高，如果更新整个图片则提供每帧的宽高即可  
第8个参数标记是否是midMap  
第9个参数标记是否在上传完毕后自动close nativeImage  
{: .prompt-tip }

在`close`函数中：  
**对所有nativeImage调用close来释放内存**

> 和普通图片材质一样，不再需要该材质后应及时调用`textureManager.release(loc)`释放资源(此时close函数会被调用)。  
如果要进一步优化性能，可以将所有不同的帧图片打包进一个png中，然后在更新的时候使用适当的参数从单个png中选择所需的矩形区域。
{: .prompt-info }

> 通过单个材质的方式实现的动态图片渲染要求所有图片的宽高必须相等(有不相等的部分应当用透明像素填充)。
如果需要渲染多种不同宽高的图片且不方便填充透明像素，则应该使用多个不同的材质实现
{: .prompt-warning }
