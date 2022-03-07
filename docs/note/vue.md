# 语法

## ``

使用``拼接字符串,参数直接${}插入

```
goToDetail(item) {
      this.$router.push({ path: `product/${item.goodsId}` })
    }
```

拼接动态方法

```
this[`exec${this.operationType}`
```



## 事件修饰符

给vue**组件绑定原生事件时候**，必须加上native ，否则会认为监听的是来自Item组件自定义的事件

如给el-input添加回车事件

```
@keyup.enter.native="handleLogin"
```



**阻止事件冒泡**

```
@click.stop
```



## 嵌套组件

自定义组件中还可以嵌套组件, 添加

```
<slot />
```

## computed高级

可和vuex配合使用与store交互

```
 computed: {
    //计算属性默认只有 getter，这里根据需求提供一个 setter：
    fixedHeader: {
      //返回vuex中的值
      get() {
        return this.$store.state.settings.fixedHeader
      },
      //设置vuex中的值
      set(val) {
        this.$store.dispatch('settings/changeSetting', {
          key: 'fixedHeader',
          value: val
        })
      }
    },
  }
```

## mixins混入对象

可用于加入与页面主要功能无关的其他功能,用于解耦

### 监听窗口变化事件

在顶层父容器或布局中进行混入以监听窗口大小做出处理

```
import store from '@/store'

const { body } = document
const WIDTH = 992 // refer to Bootstrap's responsive design

export default {
  beforeMount() {
    window.addEventListener('resize', this.$_resizeHandler)
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.$_resizeHandler)
  },
  mounted() {
    const isMobile = this.$_isMobile()
    if (isMobile) {
      store.dispatch('app/toggleDevice', 'mobile')
      store.dispatch('app/closeSideBar', { withoutAnimation: true })
    }
  },
  methods: {
    // use $_ for mixins properties
    // https://vuejs.org/v2/style-guide/index.html#Private-property-names-essential
    $_isMobile() {
      const rect = body.getBoundingClientRect()
      return rect.width - 1 < WIDTH
    },
    $_resizeHandler() {
      if (!document.hidden) {
        const isMobile = this.$_isMobile()
        store.dispatch('app/toggleDevice', isMobile ? 'mobile' : 'desktop')

        if (isMobile) {
          store.dispatch('app/closeSideBar', { withoutAnimation: true })
        }
      }
    }
  }
}

```

引入

```
import ResizeMixin from './mixin/ResizeHandler'


// 混入对象
  mixins: [ResizeMixin],
```



# 配置

## process.env

为.env文件中的内容

## proxy

```
proxyObj['/ws'] = {
    ws: true,
    target: "ws://localhost:8081"
};
proxyObj['/'] = {
    ws: false,
    target: 'http://localhost:8081',
    changeOrigin: true,
    pathRewrite: {
        '^/': ''
    }
}
module.exports = {
    devServer: {
        host: 'localhost',
        port: 8080,
        proxy: proxyObj
    }
}
```





# router

## 新标签页跳转

```
let newUrl = this.$router.resolve({
	path:'/xxx/xxx',
	query:{
		xxx:xxx,
		xxx:xxx
	},
	params:{
		xxx:Xxx,
		
	}
})
```

## 懒加载路由

```
component: (resolve) => require(['@/views/redirect'], resolve)
```

## 页面标题变化

```
router.beforeEach((to,from,next)=>{
    if(to.meta.title){
        document.title = to.meta.title;
    }
    next()
})
```





# axios

多个请求拦截器后面的先执行

```
import axios from 'axios'
import { Toast } from 'vant'
import router from '../router'
//设置请求url前缀
axios.defaults.baseURL = process.env.NODE_ENV == 'development' ? 'http://localhost:28019/api/v1' : 'http://backend-api-01.newbee.ltd/api/v1'
axios.defaults.withCredentials = true
//设置请求头
axios.defaults.headers['X-Requested-With'] = 'XMLHttpRequest'
axios.defaults.headers['token'] = localStorage.getItem('token') || ''
axios.defaults.headers.post['Content-Type'] = 'application/json'
//响应拦截器 有请求拦截器
axios.interceptors.response.use(res => {
  if (typeof res.data !== 'object') {
    Toast.fail('服务端异常！')
    return Promise.reject(res)
  }
  if (res.data.resultCode != 200) {
    if (res.data.message) Toast.fail(res.data.message)
    if (res.data.resultCode == 416) {
      router.push({ path: '/login' })
    }
    return Promise.reject(res.data)
  }

  return res.data
})

export default axios
```

使用

```
import axios from '../utils/axios'

export function createOrder(params) {
  return axios.post('/saveOrder', params);
}

```

## create二次封装

```
import axios from 'axios'
const service = axios.create({
  baseURL: process.env.VUE_APP_BASE_API,
  timeout: 10000
})
export default service
```

```
import request from '@/utils/xxx'

 request({
    url: '/login',
    method: 'post',
    data: data
  })
```

# nextTick

如果你在Vue生命周期的created()/mounted()钩子函数进行的DOM操作一定要放在Vue.nextTick()的回调函数中。原因是什么呢，原因是在created()/mounted()钩子函数执行的时候DOM 其实并未进行任何渲染，而此时进行DOM操作无异于徒劳，所以此处一定要将DOM操作的js代码放进Vue.nextTick()的回调函数中。

# 组件注册

局部:

```html
import countTo from 'vue-count-to';
export default {
  components: { countTo },
  data () {
    return {
      startVal: 0,
      endVal: 2020
    }
  }
}
```

全局:(main.js)

```js
import countTo from 'vue-count-to'
Vue.component('countTo', countTo)
```

# async/await

**async**

async 函数返回的是一个promise 对象，如果要获取到promise 返回值，我们应该用then 方法

```
async function timeout() {
    return 'hello world'
}
timeout().then(result => {
    console.log(result);
})
console.log('虽然在后面，但是我先执行');
```

**await**

它后面可以放任何表达式，不过更多的是放一个返回promise 对象的表达式。

```
function doubleAfter2seconds(num) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(2 * num)
        }, 2000);
    } )
}
```

```
async function testResult() {
    let first = await doubleAfter2seconds(30);
    let second = await doubleAfter2seconds(50);
    let third = await doubleAfter2seconds(30);
    console.log(first + second + third);
}
```

这俩关键字也可以用在生命周期函数上面,如create和mounted



# directive自定义指令

```
import store from '@/store'

export default {
  inserted(el, binding, vnode) {
    const { value } = binding
    const super_admin = "admin";
    const roles = store.getters && store.getters.roles

    if (value && value instanceof Array && value.length > 0) {
      const roleFlag = value

      const hasRole = roles.some(role => {
        return super_admin === role || roleFlag.includes(role)
      })

      if (!hasRole) {
        el.parentNode && el.parentNode.removeChild(el)
      }
    } else {
      throw new Error(`请设置角色权限标签值"`)
    }
  }
}
```



# keep-alive缓存

当组件在keep-alive内被切换时组件的**activated、deactivated**这两个生命周期钩子函数会被执行

被包裹在keep-alive中的组件的状态将会被保留，例如我们将某个列表类组件内容滑动到第100条位置，那么我们在切换到一个组件后再次切换回到该组件，该组件的位置状态依旧会保持在第100条列表处.

tip:由于目前 `keep-alive` 和 `router-view` 是强耦合的，而且查看文档和源码不难发现 `keep-alive` 的 [include](https://cn.vuejs.org/v2/api/#keep-alive)默认是优先匹配组件的 **name** ，所以在编写路由 router 和路由对应的 view component 的时候一定要确保 两者的 name 是完全一致的。(切记 name 命名时候尽量保证唯一性 切记不要和某些组件的命名重复了，不然会递归引用最后内存溢出等问题)

```
//router 路由声明
{
  path: 'config',
  component: ()=>import('@/views/system/config/index'),
  name: 'Config',
  meta: { title: '参数设置', icon: 'edit' }
}
//路由对应的view  system/config/index
export default {
  name: 'Config'
}


```



# 设置head

index.html放在src同级目录下非src

```
<!DOCTYPE html>
<html lang="">
<head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width,initial-scale=1.0">
    <!--  favicon.ico为public下面的文件 -->
    <link rel="icon" href="<%= BASE_URL %>favicon.ico">
    <title>爸爸 - 表情包专业户</title>
    <meta name="keywords" content="表情包,搞笑,斗图,图片,表情制作"/>
    <meta name="description" content="免费表情包分享制作下载平台，海量表情、随意搜索、随意编辑、原图下载。搜表情，找爸爸！"/>
    <script>
        // 在这里添加百度统计
    </script>
</head>
<body>
<noscript>
    <strong>We're sorry but <%= htmlWebpackPlugin.options.title %> doesn't work properly without JavaScript enabled.
        Please enable it to continue.</strong>
</noscript>
<div id="app"></div>
<!-- built files will be auto injected -->
</body>
</html>
```

```
import hasRole from './permission/hasRole'
import hasPermi from './permission/hasPermi'

//指令声明
const install = function(Vue) {
  Vue.directive('hasRole', hasRole)
  Vue.directive('hasPermi', hasPermi)
}

if (window.Vue) {
  window['hasRole'] = hasRole
  window['hasPermi'] = hasPermi
  Vue.use(install); // eslint-disable-line
}

export default install
```

main.js

```
import directive from './directive' //directive  权限自定义指令
Vue.use(directive) //使用自定义指令
```

使用

```
// 单个
<el-button v-hasPermi="['system:user:add']">存在权限字符串才能看到</el-button>
// 多个
<el-button v-hasPermi="['system:user:add', 'system:user:edit']">包含权限字符串才能看到</el-button>
```

# 组件

## vue2-verify

纯前端验证码组件：https://hub.fastgit.org/mizuka-wu/vue2-verify

```
          <Verify ref="loginVerifyRef" @error="error" :showButton="false" @success="success" :width="'100%'" :height="'40px'" :fontSize="'16px'" :type="2"></Verify>
          
           success(obj) {
      this.verify = true
      // 回调之后，刷新验证码
      obj.refresh()
    },
    error(obj) {
      this.verify = false
      // 回调之后，刷新验证码
      obj.refresh()
    }

```

手动触发验证

```
this.$refs.loginVerifyRef.$refs.instance.checkCode()
```



# vue/cli 3.0

## vue ui

vue ui是@vue/cli3.0增加一个可视化项目管理工具，可以运行项目、打包项目，检查等操作。对于初学者来说，可以少记一些命令

