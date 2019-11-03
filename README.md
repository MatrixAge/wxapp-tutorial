<p align="center"><a href="#" target="_blank" rel="noopener noreferrer"><img width="200" src="https://s2.ax1x.com/2019/11/03/KjVWj0.png" alt="logo"></a></p>

<p align="center">wxapp-tutorial</p>

<p align="center">微信小程序（wechat-weapp/wechat-wxapp）开发手册</p>

从工作到现在，我已经累计写了超过6个小程序，总结以及探索到了许多便于开发的解决方案，在这里分享给大家。

### 编码规范

#### 缘由

编码规范应放在首位。当一个项目需要多个人同时进行编码时，编码规范尤为重要，当我看见我编写的代码里被其他人写的其他风格的代码“污染”了时，那感觉像吃了屎一样。这就是为什么我要求团队有一个统一的代码规范，这会极大地提升代码的可读性，以及协同开发的开发效率。

不过现实是，即便开发之前已经订好了规范，总有人在不经意间去在项目里“大展拳脚”，体现自己的独特性，这个时候就需要代码review了。在国外，开发工作并不是最繁琐的，最繁琐的是代码review，Google、Microsoft这些建立在代码上的“帝国”都有一套成熟完善的代码review机制，这足以说明对于建立在代码上的“帝国”而言，编码规范在某种程度上就相当于法律，严酷的惩罚将每个开发人员的“邪念”压制住，这是我们应该了解和学习的。

#### 规范

文件命名：文件及文件夹统一使用下划线命名，比如`goods_detail.wxml`，不使用中横线的原因是其严重影响可读性。

组件命名：组件统一使用首字母驼峰命名，比如`NavBar.wxml`，使用首字母驼峰的原因是组件在页面中是一类特殊元素，需要明显的标识让它与其他便签或方法区别开来。

变量命名：变量统一使用下划线命名，比如`goods_coupon`，对于方法内的特殊私有变量，使用下划线开头，比如`const _that = this`，对于系统级别的变量，采用大写下划线命名，比如`MAX_VALUE`。

方法命名：方法统一使用驼峰命名，比如`GetUserInfo`，私有方法使用下划线开头的驼峰命名，比如`_onClose`，在wxs文件中定义的全局方法使用大写的下划线命名，比如：`FORMAT_PRICE`。

其他：在js文件中，省略语句后面分号，提升可读性。对于使用次数不超过两次的值，不单独设置变量。在方法中，对于不同类型的语句，使用空行分隔，便于阅读。CSS属性排列顺序为：绝对定位>flex定位>float定位>width/height>padding/margin>border>background>font相关>特殊属性。

更多内容请参照这篇文章：[大前端团队代码规范](https://juejin.im/post/5db919816fb9a020333c362f)

### 借助wxss-cli在小程序中使用less

#### 安装

win `npm i wxss-cli -g` 

mac `sudo npm i wxss-cli -g`

#### 使用

使用终端（CMD）进入到项目目录，执行 `wxss .`，`.`表示当前目录。即可看到less文件将被编译成同名的wxss文件。

![img][https://s2.ax1x.com/2019/11/03/KjE3Js.png]

![img][https://s2.ax1x.com/2019/11/03/KjE1ij.png]

### 使用CSS原生变量

#### 场景

有时候我们与一些不适用less进行开发的小伙伴进行协同，需要统一变量，这个时候CSS原生变量就派上了用场。

#### 定义

app.wxss

```
page{
    --color_main:#333333;
    --color_sub:#555555;
}
```

#### 使用

index.wxss

```
@import '../app.wxss';

.section .title{
    color:var(--color_main);
}

.section .content{
    color:var(--color_sub);
}
```

### 使用defineProperty定义app.globalData 构建全局数据中心

#### 缘由

有时，我们会遇到一些很特殊的需求，比如需要在自定义的tabbar上与页面进行交互，而微信的自定义tabbar又没法使用triggerEvent（在页面里不需要引用tabbar），这个时候就需要借助`const app = getApp()`来实现类似于数据中心的功能。

现在的需求是，进入到购物车页面，选择完商品之后需要在tabbar上显示总的价格，然后点击tabbar上的付款按钮。那我们首先得拿到实时的购物车选中商品的数据，也就是做我们得监听购物车选中了哪些商品，同时在tabbar里面做出响应，这个时候，熟悉JS的同学就会想到defineProperty这个方法，它允许我们定义一个变量，同时提供get、set等钩子函数，类似于C#中的getter、setter方法。

首先我们在app里定义这个变量，同时构建这个“数据中心”所需的“基础设施”：

app.js

```
App({
    globalData:{
        orders:[]
    },
    
    // watcher start
    watchCallBack: {},
    watchingKeys: [],
    initWatcher () {
    	this.globalData$ = Object.assign({}, this.globalData)
    },
    setGlobalData (obj) {
    	Object.keys(obj).map((key) => {
    		this.globalData[key] = obj[key]
    	})
    },
    watch$ (key, cb) {
    	this.watchCallBack = Object.assign({}, this.watchCallBack, {
    		[key]: this.watchCallBack[key] || []
    	})
    
    	this.watchCallBack[key].push(cb)
    
    	if (!this.watchingKeys.find((x) => x === key)) {
    		const that = this
    		this.watchingKeys.push(key)
    		Object.defineProperty(this.globalData, key, {
    			configurable: true,
    			enumerable: true,
    			set: function (val){
    				const old = that.globalData$[key]
    				that.globalData$[key] = val
    				that.watchCallBack[key].map((func) => func(val, old))
    			},
    			get: function (){
    				return that.globalData$[key]
    			}
    		})
    	}
    },
    // watcher end
    
)

```

cart.js

```
onChangeOrders() {
    const _that = this
    
	app.setGlobalData({
		orders: _that.data.orders
	})
}
```

tabbar.js

```
watchOrders () {
	const _that = this

	app.watch$('orders', (new_val) => {
		if (new_val.length) {
			_that.setData({
				orders: new_val
			})
		}
	})
},
```

了解更多请看这篇文章：[50行代码监听watch小程序的globalData](https://www.cnblogs.com/BestMePeng/p/xcx_watch_globaldata.html)

### 使用wxs文件定义模板filter

#### 缘由

写过Vue的同学知道，在Vue里有一个特别方便的东西，就是filter，它可以很方便地处理模板变量，尤其是在循环数组时，但小程序并没有直接提供相关功能，其实我们可以借助小程序的wxs来实现（目前wxs对ES6的支持有限，许多高级特性都无法使用，比如Object、Array）。

#### 实现

app.wxs

```
module.exports = {
	FORMAT_PRICE: function (price){
		var value = (price / 100).toFixed(2)

		return value
	},
	FORMAT_ORDER_NUMBER: function (number){
		var value = number.slice(0, 12)

		return value
	},
	FORMAT_JSON_TO_STRING: function (string){
		return JSON.parse(string)
	}
}
```

#### 使用

cart.wxml

```
<wxs
      src="../app.wxs"
      module="app"
/>

<text class="price">{{app.FORMAT_PRICE(item.price)}}</text>
```

### 使用atom.css配合vs code插件IntelliSense for CSS class names in HTML快速出页面

#### 缘由

由于小程序体积限制（2M），在小程序中基本上无法使用那种大而全UI框架，而这个时候CSS框架似乎是更好的选择，这里推荐使用[atom.css](https://github.com/MatrixAge/atom.css)。

#### 安装

`npm i @verts/atom.css --save`

#### 使用

然后复制里面的`atom-miniapp.min.wxss`文件到项目目录即可。如果要在less中使用，将`atom-miniapp.min.wxss`更改为`atom-miniapp.min.less`即可。配合vscode使用时需要安装`IntelliSense for CSS class names in HTML`，并将atom.css项目文件夹引入到vscode中，让上述插件将atom.css的所有class缓存到vscode中（wxss 插件无法识别，故无法缓存）。

#### 效果

![img_3][https://s2.ax1x.com/2019/11/03/KjE8Wn.png]

### 使用async包装wx.request 简单实现拦截器的功能

#### 实现

request.js

```
const request = (url, method, data, header) => {
	return new Promise((resolve, reject) => {
		wx.request({
			url: url,
			method: method,
			data: data,
			header: Object.assign(
				{ token: wx.getStorageSync('token') },
				header
			),
			success (res) {
			    //拦截器相关逻辑
			    if(res.code==='200'){
			        resolve(res.data)
			    }
			},
			fail (error) {
				reject(error)
			}
		})
	})
}

export const get = async (url, data) => {
	return request(url, 'get', data)
}

export const post = async (url, data, header) => {
	return request(url, 'post', data, header)
}

```

#### 使用

```
import API from '../../utils/api'
import { get } from '../../utils/request'

export const Service_getGoodsDetail = (data) => get(API.API_getGoodsDetail, data)
```

### 集中管理api接口

#### 缘由 

在项目进行的过程中，由于需求随时会产生变化，所以接口也可能会变化，这个时候就需要统一管理和配置接口，并保持接口的“无状态”，便于后期开发和维护。API命名格式为：`API_[methodtype][Someone][Do][Something]`

#### 实现

api.js

```
//线上地址
// const API_BASE_URL = 'https://api.***.com'

//测试地址
const API_BASE_URL = 'https://test.***.com'

//开发地址
// const API_BASE_URL = 'http://***:8080'

//登录
export const API_postUserLogin = API_BASE_URL + '/***/login'

//获取 用户信息
export const API_getUserInfo = API_BASE_URL + '/***/userInfo'

const API = {
    API_postUserLogin,
    API_getUserInfo,
}

export default API
```

### 使用mark对长列表做事件代理

#### 缘由

过去我们使用Jquery，很容易通过on方法实现列表中的子项的操作进行代理操作，我们称之为事件委托。但在小程序中如何实现这一点呢？除了使用`data`-，小程序提供给了我们一个比`data-`更加好用的方案——mark。

#### 使用

coupon.wxml

```
 <view
    class="coupon_items w_100 border_box flex flex_column"
    catchtap="onSelectCoupon"
>
    <view
          class="coupon_item w_100 border_box flex relative"
          wx:for="{{coupon_list}}"
          wx:key="{{index}}"
          mark:index="{{index}}"
    >
        ***
    </view>
</view>
```

coupon.js

```
onSelectCoupon(e){
    const _that = this
    const { index } = e.mark
    
    if(index){
        _that.setData(
            {
                select_coupon_index:index
            }
        )
    }
}
```

### 在setData之前对目标数据进行复杂操作

```
const _that = this
const { index } = e.detail

_that.data.coupon_list[index].selected = true
_that.data.coupon_list[index].text_style = 'bold'

_that.setData(
    {
        coupon_list:_that.data.coupon_list
    }
)
```

### 使用recycle-view进行长列表渲染优化

#### 缘由

之前做过一个新闻+购物的小程序，由于首页滚动列表的图片内容比较多，滚动加载很多条之后就会出现卡顿甚至直接卡出微信的情况，后来通过使用一些第三方的长列表优化插件优化了一下，至少不会直接卡出微信了，微信现在有一个官方的长列表优化方案，那就是revcycle-view，使用recycle-view能够极大地节省内存，同时提升用户体验，在angular7中，该功能已被集成到angular官方sdk中，而在react中，也有着很多虚拟滚动的方案，Facebook的Instagram Web端的PWA应用很多地方都用到了虚拟滚动。

#### 使用

index.wxml

```
<recycle-view
            id="chosen"
            class="chosen w_100 border_box flex justify_between flex_wrap"
            batch="{{batchSetRecycleData}}"
            scroll-y="{{true}}"
            scroll-with-animation="{{true}}"
            lower-threshold="{{100}}"
            bindscrolltolower="onScrollToLower"
            enable-back-to-top="{{true}}"
            scroll-top="{{scroll_top}}"
            bindscroll="onScroll"
      >
    <view
          class="w_100 border_box flex flex_column"
          slot="before"
    >
        ***
    </view>
    <view class="goods_card_items w_100 border_box flex justify_between flex_wrap">
          <recycle-item
                class="goods_card_wrap"
                wx:for="{{recycleList}}"
                wx:key="{{item.__index__}}"
          >
                <GoodsCard
                      class="goods_card"
                      goods_id="{{item.goods_id}}"
                ></GoodsCard>
          </recycle-item>
    </view>
</recycle-view>
```

index.js

```
data:{}
recycle_view_context:{},
createRecycleView () {
	const _that = this

	const ctx = createRecycleContext({
		id: 'chosen',
		dataKey: 'recycleList',
		page: _that,
		itemSize: {
			width: '100%',
			height: 350
		}
	})

	_that.recycle_view_context = ctx
},
```

追加数据
```
getGoodsData () {
	const _that = this

	Service_getGoodsData.then((res) => {
		if (res.data) {
			_that.recycle_view_context.append(res.data.goods_list)
		}
	})
},
```

### 使用watch来实现dialog动效

#### 缘由

之前写一个商城的时候，研究淘宝的商品属性选择弹窗的动效是一个什么过程，然后如何实现。过程如下：点击选择属性/购买/加入购物车按钮 => 蒙版层占满整个屏幕，然后渐渐显现出来，与此同时，下方的窗口从底部渐渐滑出。OK，其实这个过程很简单，点击按钮的时候，弹窗（占满整个屏幕）其实以及出现了，但是要等待蒙版层的背景颜色从transparent变为rgba(0, 0, 0, 0.6)，同时也在等待弹窗内容从transform:translate(-100%)变为transform:translate(0)。这个时候需要用到小程序官方开源的一个插件：[miniprogram-computed](https://developers.weixin.qq.com/miniprogram/dev/extended/utils/computed.html)

#### 实现

Dialog.wxml
```
<view
      class="fixed_wrap"
      wx:if="{{is_show}}"
      catchtouchmove="onStopPageScroll"
>
      <view class="dialog_wrap">
            <view
                  class="mask"
                  style="background-color: {{bg_modal}}"
                  catchtap="onClose"
            ></view>
            <view
                  class="dialog_absolute_wrap"
                  style="transform:{{position_dialog}}"
            >
                  <view
                        class="dialog"
                        style="background-color: {{is_show_bg?'white':'transparent'}}"
                  >
                        <image
                              class="img_close"
                              src="../../assets/images/icon_close.svg"
                              mode="widthFix"
                              bindtap="onClose"
                              wx:if="{{is_show_close}}"
                        ></image>
                        <slot></slot>
                  </view>
            </view>
      </view>
</view>
```

dialog.js
```
import computedBehavior from 'miniprogram-computed'

Component({
	behaviors: [
		computedBehavior
	],
	properties: {
		id: {
			type: String
		},
		is_show_dialog: {
			type: Boolean,
			value: false
		},
		is_show_close: {
			type: Boolean,
			value: true
		},
		is_show_bg: {
			type: Boolean,
			value: true
		}
	},
	data: {
		is_show: false,
		bg_modal: 'transparent',
		position_dialog: 'translateY(120%)'
	},
	watch: {
		is_show_dialog: function (new_val){
			const _that = this

			if (new_val) {
				_that.setData({
					is_show: true
				})

				setTimeout(() => {
					_that.setData({
						bg_modal: 'rgba(0, 0, 0, 0.6)',
						position_dialog: 'translateY(0)'
					})
				}, 0)
			}
			else {
				_that.setData({
					bg_modal: 'transparent',
					position_dialog: 'translateY(120%)'
				})

				setTimeout(() => {
					_that.setData({
						is_show: false
					})
				}, 300)
			}
		}
	},
	methods: {
		onClose () {
			const _that = this

			_that.setData({ is_show_dialog: false })
		},
		onTapDialog (e) {
			const _that = this

			_that.triggerEvent('OnBottomDialog', { id: _that.data.id, event: e })
		},
		onStopPageScroll () {}
	}
})
```

dialog.less
```
@import '../../theme/vars.less';

.fixed_wrap {
      position: fixed;
      top: 0;
      left: 0;
      z-index: 10000;
      width: 100vw;
      height: 100vh;
     
}

.dialog_wrap {
      position: relative;
      width: 100%;
      height: 100%;

      .mask {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            transition: all ease 0.3s;
      }

      .dialog_absolute_wrap {
            position: absolute;
            bottom: 0;
            left: 0;
            width: 100%;
            transition: all ease 0.3s;

            .dialog {
                  position: relative;
                  display: flex;
                  flex-direction: column;
                  width: 100%;
                  box-sizing: border-box;
                  background-color: white;
                  border-top-left-radius: @radius_normal;
                  border-top-right-radius: @radius_normal;

                  .img_close {
                        position: absolute;
                        top: 10rpx;
                        right: 10rpx;
                        z-index: 1;
                        width: 40rpx;
                        height: 40rpx;
                        padding: 20rpx;
                        opacity: 0.3;
                  }
            }
      }
}
```

### 自定义顶部NavBar

#### 缘由

现在有很多商城app都会有在顶部NavBar上放各种“东西”比如搜索框的需求，但是NavBar上的胶囊又没办法自定义，这就导致NavBar上的搜索框或是其他元素会与胶囊错位，看起来效果很差，详情可看“网易严选小程序”，但是其实是有办法做到顶部NavBar完美布局的，这里就要用到一个获取胶囊定位以及尺寸的api [wx.getMenuButtonBoundingClientRect()
](https://developers.weixin.qq.com/miniprogram/dev/api/ui/menu/wx.getMenuButtonBoundingClientRect.html)，通过这个api，然后使用`res.statusBarHeight`这个变量，我们可以做很多事情。不过经过测试，目前已知在小米的一些型号的全面屏手机上，高度会有略微的偏移，需要做一定的适配。

```
wx.getSystemInfo({
	success(res){
	    console.log(res.statusBarHeight)
	},
})
```

#### 实现

具体实现我就不写了，等有时间抽空把组件重构一次后再分享给大家，下面写部分关键代码：

```
//获取设备顶部 状态栏高度 和 顶部标题栏高度
export const GetDeviceBarHeight = () => {
	let statusBarHeight
	let titleBarHeight

	wx.getSystemInfo({
		success(res){
			let totalTopHeight = 68

			if (res.model.indexOf('iPhone X') !== -1) {
				totalTopHeight = 88
			}
			else if (res.model.indexOf('iPhone') !== -1) {
				totalTopHeight = 64
			}

			statusBarHeight = res.statusBarHeight
			titleBarHeight = totalTopHeight - res.statusBarHeight
		},
		failure () {
			statusBarHeight = 0
			titleBarHeight = 0
		}
	})

	return {
		statusBarHeight: statusBarHeight,
		titleBarHeight: titleBarHeight
	}
}

data: {
        statusBarHeight: app.globalData.statusBarHeight + 'px',
		titleBarHeight: app.globalData.titleBarHeight + 'px',
		navigationBarHeight: app.globalData.statusBarHeight + 44 + 'px',

		height: 0,
		top: 0
},

//动态设定 顶部操作按钮的高度和位置
setOptionsHeight () {
	const _that = this
	const { top, height } = wx.getMenuButtonBoundingClientRect()

	_that.setData({
		height: height - 1,
		top: top + 2
	})
}
```

### 小程序分包加载

但小程序的大小超过2M之后可以采用分包加载的方式加载页面，具体使用方式看官方文档[小程序分包加载](https://developers.weixin.qq.com/miniprogram/dev/framework/subpackages/basic.html)

### 最后

后续我会写一些单独的小程序组件并开源出来，不同于有赞的小程序库，他们提供的是一辆长得像淘宝的“改装车”，而我开源出来的是一些，跟谁都长的不像，想怎么改就怎么改的“改装车零件”。

我在github开源了一个[小程序开发手册](https://github.com/MatrixAge/wxapp-tutorial)，后续更多内容将在这个开源项目上更新，大家可以分享给开发小程序的同事，让他们少踩一点坑，大家少加一点班。

周末愉快。






























[img_1]:data:image/png;base64,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

[img_2]:data:image/png;base64,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

[img_3]:data:image/png;base64,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
