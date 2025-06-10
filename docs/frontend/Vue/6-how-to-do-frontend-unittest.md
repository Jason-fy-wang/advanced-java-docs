---
tags:
  - Vue
  - unittest
  - test
  - frontend
---
在编写代码时, 单元测试是必不可少的部分, unittest 能增强programmer的信心, 也能提高代码的鲁棒性.  那么针对前端代码如何编写unit test 呢?

环境:
```
vite + vue3 + element-plus
```


#### step 1: install dependency

```
cnpm install --save-dev @pinia/testing @testing-library/jest-dom @testing-library/vue  happy-dom jsdom vitest @vue/test-utils
```

目录结构:
```shell
LDAP
|__src
|____components
|_____________Home.vue
|____pages
|__test
|_____components
|______________Home.spec.js
...
```

#### step 2: add configuration for vitest
```javescript
// vite.config.mjs
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import path from "node:path"
import vueJsx from '@vitejs/plugin-vue-jsx'

export default defineConfig({
  resolve: {
    alias: {
      "@": path.join(__dirname, "./src")
    }
  },
  server: {
    host: true
  },
  plugins: [
    vue(),
    vueJsx()
  ],
  base: "./",
  // testing configuration
  test: {
    globals: true,
    // 渲染环境
    environment: "happy-dom",  // happy-dom  jsdom  node
    // 可以设置setup 脚本
    setupFiles: "./test/setup.js",
    css: true,
    coverage: {
      provider: "istanbul", //c8 v8
      reporter: ["text", "json", "html"],  // 单元测试报告 类型
      exclude: [
        "node_modules/",
        "src/electron/",
      ],
    },
  },
})

```

环境配置脚本:
```js
import {cleanup} from '@testing-library/vue'
import {expect, afterEach, beforeEach} from 'vitest'
import '@testing-library/jest-dom'
import {setActivePinia, createPinia} from 'pinia'
import 'element-plus/dist/index.css'

expect.extend({})
beforeEach(() => {
  // setup the Pinia store before each test
  const pinia = createPinia()
  setActivePinia(pinia)
})

afterEach(() => {
  // cleanup the DOM after each test
  cleanup()
})

// reset the localStorage after each test  
afterEach(() => {
  localStorage.clear()
})
```

#### step 3:  编写单元测试
```js
// Home.vue
<template>
    <div>
        <h1>Ldapmanager</h1>
    </div>
</template>
<script setup name="Home">
</script>
<style lang="less" scoped>
</style>
```

```js
// Home.spec.js
import {expect, describe, it} from 'vitest'
import {mount,shallowMount} from '@vue/test-utils'
// 
describe('Home.vue', () => {
    it('render properly', async () => {
        // 引入组件
        const Home = (await import('@/components/Home.vue')).default
        // 渲染组件
        const wrapper = mount(Home, ()=>{
        })
        expect(wrapper.exists()).toBeTruthy()
        expect(wrapper.find('h1').text()).toBe('Ldapmanager')
    })

    it('shadow render', async ()=> {
        const Home = (await import('@/components/Home.vue')).default
        // shallowMount 不会渲染子组件, 轻度渲染
        const wrapper = shallowMount(Home, {
            props: {
                msg: 'Ldapmanager'
            }
        })
        // 条件判断
        expect(wrapper.exists()).toBeTruthy()
        expect(wrapper.find('h1').text()).toBe('Ldapmanager')
    })
})
```

#### step 4: 添加测试命令
```json
  "scripts": {
...
    "test": "vitest run --reporter=verbose",
    "test:coverage": "vitest run --coverage"
  },
```



> 注意:
> 当通过const wrapper = mount(Home, ()=>{}) 挂在组件后, 可以通过console.log("html:", wrapper.html()) 打印出最终选择的页面.  因为本环境使用的element-ui, 故Home组件中所有的element-ui组件都会被正常渲染.  如果发现 element-ui组件没有被渲染, 那么要确保: 1. element-ui 被正确注册了. 2. 删除 node_modules 重新install并进行渲染

注册plugin:
```js
import {flushPromises, mount} from '@vue/test-utils'
import {expect, describe, it, beforeEach,vi} from 'vitest'
import ElementPlus from 'element-plus'
import {createTestingPinia} from '@pinia/testing'
import { createRouter,createWebHistory } from 'vue-router'
import Login from '@/components/Login.vue'

// Create mock router
const router = createRouter({
    history: createWebHistory(),
    routes: [
        {
            path: '/',
            name: 'login',
            component: Login
        },
        {
            path: '/home',
            name: 'home',
            component: { template: '<div>Home</div>' }
        }
    ]
})

  
// mock login function
const loginMock = vi.fn().mockResolvedValue(true)
vi.mock('@/pages/api/useLdap', () => ({
    default: () => ({
        login: loginMock
    })
}))
  
describe('Login.vue',  () => {
    let pinia, wrapper
    beforeEach(() => {
        pinia = createTestingPinia({
            initialState: {
                objectattribute: {
                    attributes: {},
                    loginState: false,
                    loginName: "anonymous",
                    displayName: "anonymous",
                    domainName: "anonymous",
                }
            },
            stubActions: false
        })

        wrapper = mount(Login, {
            global: {
                plugins: [
                // 把 elementPlus, pinia, router组件注册
                    ElementPlus,
                    pinia,
                    router
                ]
            }
        })
    })

    afterEach(() => {
        vi.clearAllMocks()
        wrapper.unmount()
    })
```



