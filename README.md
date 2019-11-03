<p align="center"><a href="#" target="_blank" rel="noopener noreferrer"><img width="200" src="https://s2.ax1x.com/2019/11/03/KjVWj0.png" alt="logo"></a></p>

<p align="center">wxapp-tutorial</p>

<p align="center">微信小程序（wechat-weapp/wechat-wxapp）开发手册</p>

[小程序开发手册 github](https://github.com/MatrixAge/wxapp-tutorial)

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

![img_1](https://s2.ax1x.com/2019/11/03/KjE3Js.png)

![img_2](https://s2.ax1x.com/2019/11/03/KjE1ij.png)

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

![img_3](https://s2.ax1x.com/2019/11/03/KjE8Wn.png)

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
