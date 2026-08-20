# 五. React

> **鸣谢**：本文参考了「栗子」的教程，是我学习后二次整理的内容，在此致谢；若有错漏，皆因我个人水平有限，与「栗子」无关。

## 1. React 基础

react 是前端三大框架之一

* 没有 vue 基础反而更好，两者思想不太一样，别用 vue 的习惯学 react
* 需要 js 基础（对应视频 19-58）
* 需要 ts 基础（对应视频 110-116）
* 本笔记用更流行的【函数式组件 + hooks】方式讲解



### 1) 环境准备

#### 创建项目

先用 react 脚手架创建项目

```cmd
npx create-react-app client --template typescript
```

* client 是项目名
* 目前 react 版本是 18.x

> **P.S.** create-react-app 已被 React 官方弃用，新项目建议改用 Vite 创建（`npm init vite@latest`，选择 react + typescript 模板）。本章沿用 CRA 是因为课程代码基于它，这些知识在 Vite 项目里同样适用

运行项目

```cmd
cd client
npm start
```

* 会自动打开浏览器，默认监听 3000 端口

![image-20221001110328233](imgs/image-20221001110328233.png)



#### 修改端口

在项目根目录下新建 .env.development，用来定义开发环境的环境变量

```properties
PORT=7070
```

重启项目，端口就变成了 7070

* 参考文档：[Advanced Configuration | Create React App (create-react-app.dev)](https://create-react-app.dev/docs/advanced-configuration/)



#### 浏览器插件

插件地址 [New React Developer Tools – React Blog (reactjs.org)](https://reactjs.org/blog/2015/09/02/new-react-developer-tools.html#installation)

![image-20221004105110150](imgs/image-20221004105110150.png)

#### VSCode

推荐安装 Prettier 代码格式化插件

![image-20221004090816142](imgs/image-20221004090816142.png)



### 2) 入门案例

#### Hello

编写一个 src/pages/Hello.tsx 组件

```tsx
export default function Hello()  {
  return <h3>Hello, World!</h3>
}
```

* 组件中使用了 jsx 语法，即在 js 中直接使用 html 标签或组件标签
* 函数式组件必须返回标签片段

在入口文件 index.tsx 中引入组件

```tsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import './index.css'
import reportWebVitals from './reportWebVitals'
// 1. 引入组件
import Hello from './pages/Hello'

const root = ReactDOM.createRoot(document.getElementById('root') as HTMLElement)
root.render(
  <React.StrictMode>
    {/* 2. 将原来的 <App/> 改为 <Hello></Hello> */}
    <Hello></Hello>
  </React.StrictMode>
)

// If you want to start measuring performance in your app, pass a function
// to log results (for example: reportWebVitals(console.log))
// or send to an analytics endpoint. Learn more: https://bit.ly/CRA-vitals
reportWebVitals()
```



将欢迎词作为属性传递给组件

```tsx
<Hello msg='你好'></Hello>
```

* 字符串值，可以直接使用双引号赋值
* 其它类型的值，用 `{值}`

组件则改成

```tsx
export default function Hello(props: { msg: string }) {
  return <h3>{props.msg}</h3>
}
```



#### jsx 原理

```tsx
export default function Hello(props: { msg: string }) {
  return <h3>{props.msg}</h3>
}
```

在 v17 之前，其实相当于

```tsx
import { createElement } from "react";
export default function Hello(props: {msg: string}) {
  return createElement('h3', null, `${props.msg}`)
}
```



### 3) 人物卡片案例

样式已准备好 /src/css/P1.css

```css
#root {
  display: flex;
  width: 100vw;
  height: 100vh;
  justify-content: center;
  align-items: center;
}

div.student {
  flex-shrink: 0;
  flex-grow: 0;
  position: relative;
  width: 128px;
  height: 330px;
  /* font-family: '华文行楷'; */
  font-size: 14px;
  text-align: center;
  margin: 20px;
  display: flex;
  justify-content: flex-start;
  background-color: #7591AD;
  align-items: center;
  flex-direction: column;
  border-radius: 5px;
  box-shadow: 0 0 8px #2c2c2c;
  color: #e8f6fd;
}

.photo {
  position: absolute;
  width: 100%;
  height: 100%;
  top: 0;
  border-radius: 0%;
  overflow: hidden;
  transition: 0.3s;
  border-radius: 5px;
}

.photo img {
  width: 100%;
  height: 100%;
  /* object-fit: scale-down; */
  object-fit: cover;
}

.photo::before {
  position: absolute;
  content: '';
  width: 100%;
  height: 100%;
  background-image: linear-gradient(to top, #333, transparent);
}

div.student h2 {
  position: absolute;
  font-size: 20px;
  width: 100%;
  height: 68px;
  font-weight: normal;
  text-align: center;
  margin: 0;
  line-height: 68px;
  visibility: hidden;
}

h2::before {
  position: absolute;
  top: 0;
  left: 0;
  content: '';
  width: 100%;
  height: 68px;
  background-color: rgba(0, 0, 0, 0.3);
  border-top-left-radius: 5px;
  border-top-right-radius: 5px;  
}

div.student h1 {
  position: absolute;
  top: 250px;
  font-size: 22px;
  margin: 0;
  transition: 0.3s;
  font-weight: normal;
}

div.student p {
  margin-top: 300px;
  width: 80%;
  font-weight: normal;
  text-align: center;
  padding-bottom: 5px;
  border-bottom: 1px solid #8ea2b8;
}

.student:hover .photo::before {
  display: none;
}

.student:hover .photo {
  width: 90px;
  height: 90px;
  top: 90px;
  border-radius: 50%;
  box-shadow: 0 0 15px #111;
}

.student:hover img {
  object-position: 50% 0%;
}

.student:hover h1 {
  position: absolute;
  top: 190px;
  width: 40px;
}

div.student:hover h2 {
  visibility: visible;
}
```

类型 /src/model/Student.ts

```typescript
export interface Student {
  id: number,
  name: string,
  sex: string,
  age: number,
  photo: string
}
```

组件 /src/pages/P1.tsx

```tsx
import { Student } from '../model/Student'
import '../css/P1.css'
export default function P1(props: { student: Student }) {
  return (
    <div className='student'>
      <div className='photo'>
        <img src={props.student.photo}/>
      </div>
      <h1>{props.student.name}</h1>
      <h2>{props.student.id}</h2>      
      <p>性别 {props.student.sex} 年龄 {props.student.age}</p>
    </div>
  )
}
```

使用组件

```tsx
const stu1 = { id: 1, name: '风清扬', sex: '男', age: 99, photo: '/imgs/1.png' }
const stu2 = { id: 2, name: '玮哥', sex: '男', age: 20, photo: '/imgs/2.png' }
const stu3 = { id: 3, name: '长宏', sex: '男', age: 30, photo: '/imgs/3.png'}

<P1 student={stu1}></P1>
<P1 student={stu2}></P1>
<P1 student={stu3}></P1>
```



#### 路径

* src 下的资源，要用相对路径引入
* public 下的资源，记得 / 代表路径的起点



#### 标签命名

* 组件标签必须用大驼峰命名
* 普通 html 标签必须用小写命名



#### 事件处理

```tsx
import { Student } from '../model/Student'
import '../css/P1.css'
export default function P1(props: { student: Student }) {
    
  function handleClick(e : React.MouseEvent){
    console.log(props.student)
    console.log(e)
  }
  
  return (
    <div className='student'>
      <div className='photo' onClick={handleClick}>
        <img src={props.student.photo}/>
      </div>
      <h1>{props.student.name}</h1>
      <h2>{props.student.id}</h2>
      <p>性别 {props.student.sex} 年龄 {props.student.age}</p>
    </div>
  )
}
```

* 事件以小驼峰命名
* 事件处理函数可以有一个事件对象参数，可以获取事件相关信息



#### 列表 & Key

```tsx
import { Student } from '../model/Student'
import P1 from './P1'

export default function P2(props: { students: Student[] }) {
  return (
    <>
      {props.students.map((s) => ( <P1 student={s} key={s.id}></P1> ))}
    </>
  )
}
```

* key 在循环时是必须的，否则会有 warning

也可以这样写

```tsx
import { Student } from '../model/Student'
import P1 from './P1'

export default function P2(props: { students: Student[] }) {
  const list = props.students.map((s) => <P1 student={s} key={s.id}></P1>)
  return <>{list}</>
}
```



使用组件

```tsx
const stu1 = { id: 1, name: '风清扬', sex: '男', age: 99, photo: '/1.png' }
const stu2 = { id: 2, name: '玮哥', sex: '男', age: 45, photo: '/2.png' }
const stu3 = { id: 3, name: '长宏', sex: '男', age: 45, photo: '/3.png'}

<P2 students={[stu1,stu2,stu3]}></P2>
```



#### 条件渲染

P1 改成

```tsx
import { Student } from '../model/Student'
import '../css/P1.css'
export default function P1(props: { student: Student; hideAge?: boolean }) {
  function handleClick() {
    console.log(props.student)
  }

  const ageFragment = !props.hideAge && <span>年龄 {props.student.age}</span>

  return (
    <div className='student'>
      <div className='photo' onClick={handleClick}>
        <img src={props.student.photo} />
      </div>
      <h1>{props.student.name}</h1>
      <h2>{props.student.id}</h2>
      <p>
        性别 {props.student.sex} {ageFragment}
      </p>
    </div>
  )
}
```

* 子元素如果是布尔值，nullish，不会渲染



P2 改成

```tsx
import { Student } from '../model/Student'
import P1 from './P1'

export default function P2(props: { students: Student[]; hideAge?: boolean }) {
  const list = props.students.map((s) => (
    <P1 student={s} hideAge={props.hideAge} key={s.id}></P1>
  ))
  return <>{list}</>
}
```



使用组件

```tsx
const stu1 = { id: 1, name: '风清扬', sex: '男', age: 99, photo: '/1.png' }
const stu2 = { id: 2, name: '玮哥', sex: '男', age: 45, photo: '/2.png' }
const stu3 = { id: 3, name: '长宏', sex: '男', age: 45, photo: '/3.png'}

<P2 students={[stu1,stu2,stu3]} hideAge={true}></P2>
```



#### 参数解构

以 P1 组件为例

```tsx
import { Student } from '../model/Student'
import '../css/P1.css'
export default function P1
({ student, hideAge = false }: { student: Student, hideAge?: boolean }) {
  
  function handleClick() {
    console.log(student)
  }

  const ageFragment = !hideAge && <span>年龄 {student.age}</span>

  return (
    <div className='student'>
      <div className='photo' onClick={handleClick}>
        <img src={student.photo} />
      </div>
      <h1>{student.name}</h1>
      <h2>{student.id}</h2>
      <p>
        性别 {student.sex} {ageFragment}
      </p>
    </div>
  )
}
```

* 可以利用解构赋值语句，让 props 的使用更为简单
* 对象解构赋值还有一个额外的好处，给属性赋默认值



使用组件

```tsx
const stu1 = { id: 1, name: '风清扬', sex: '男', age: 99, photo: '/1.png' }

<P1 student={stu1}></P1>
```



### 4) 处理变化的数据

入门案例侧重数据展示，没涉及数据变动；接下来看看 react 怎么处理数据变化



#### axios

先看 axios，它负责发请求、收响应，从服务器拿真实数据

安装

```cmd
npm install axios
```

定义组件

```tsx
import axios from 'axios'
export default function P4({ id }: { id: number }) {
  async function updateStudent() {
    const resp = await axios.get(`http://localhost:8080/api/students/${id}`)
    console.log(resp.data.data)
  }

  updateStudent()

  return <></>
}
```

* 其中 /api/students/${id} 是提前备好的后端服务 api，会延迟 2s 返回结果

使用组件

```tsx
<P4 id={1}></P4>
```

在控制台上打印

```
{
    "id": 1,
    "name": "宋远桥",
    "sex": "男",
    "age": 40
}
```

当属性变化时，会重新触发 P4 组件执行，比如把 id 从 1 改成 2

![image-20221005160308705](imgs/image-20221005160308705.png)

执行流程

* 首次调用函数组件，返回的 jsx 代码会被渲染成【虚拟 dom 节点】（也称 Fiber 节点）
  * 根据【虚拟 dom 节点】会生成【真实 dom 节点】，由浏览器显示出来
* 当函数组件的 props 或 state 发生变化时，才会**重新**调用函数组件，返回 jsx
  * jsx 与上次的【虚拟 dom 节点】对比
    * 如果没变化，复用上次的节点
    * 有变化，创建新的【虚拟 dom 节点】替换掉上次的节点
* 严格模式会触发两次渲染，为避免干扰，先注释掉 index.tsx 里的 `<React.StrictMode>`



#### 状态

先看个例子：能不能把服务器返回的数据显示到页面上

```tsx
import axios from 'axios'
let count = 0
export default function P5(props: { id: number }) {
  
  function getTime() {
    const d = new Date()
    return d.getHours() + ':' + d.getMinutes() + ':' + d.getSeconds()
  }
  
  async function updateStudent() {
    const resp = await axios.get(
      `http://localhost:8080/api/students/${props.id}`
    )
    Object.assign(student, resp.data.data)
    console.log(current, student, getTime())
  }

  const current = count++
  let student = { name: 'xx' }
  console.log(current, student, getTime())
  updateStudent()
  
  return <h3>姓名是：{student.name}</h3>
}
```

* count 是一个全局变量，记录 P5 函数第几次被调用

执行后，控制台输出

```
0 {name: 'xx'} '16:22:16'
0 {id: 1, name: '宋远桥', sex: '男', age: 40} '16:22:18'
```

此时页面仍显示 `姓名是：xx`



那改成修改 props 的 id 呢？进开发工具把 id 从 1 改成 2，控制台输出

```tsx
1 {name: 'xx'} '16:24:0'
1 {id: 2, name: '俞莲舟', sex: '男', age: 38} '16:24:2'
```

此时页面仍显示 `姓名是：xx`



为什么页面没显示两秒后更新的值？

* 第一次：页面显示的是 P5 函数的返回结果，这时 student.name 还没被更新成宋远桥，所以显示 xx
  * 虽然 2s 后数据更新了，但此时并未触发函数执行，页面不变
* 第二次：虽然 props 修改触发了函数重新执行，但函数一重跑，里面的 student 又被赋成 `{ name: 'xx' }`，页面还是 xx
  * 2s 后数据更新，跟第一次一样，并未重新触发函数执行，页面不变



结论：

* 函数是无状态的，执行完它内部的数据都不会保留
* 想让函数有状态，就要用 useState 把数据存到函数之外，这些数据就叫状态



#### useState

```tsx
import axios from 'axios'
import { useState } from 'react'
import { Student } from '../model/Student'
let count = 0
export default function P5(props: { id: number }) {

  // ...

  async function updateStudent() {
    const resp = await axios.get(
      `http://localhost:8080/api/students/${props.id}`
    )
    Object.assign(student, resp.data.data)
    console.log(current, student, getTime())
  }

  const current = count++
  let [student] = useState<Student>({ name: 'xx' })
  console.log(current, student, getTime())
  updateStudent()

  return <h3>姓名是：{student.name}</h3>
}
```

接下来用 setXXX 方法更新 State

```tsx
import axios from 'axios'
import { useState } from 'react'
import { Student } from '../model/Student'
export default function P5(props: { id: number }) {
  async function updateStudent() {
    const resp = await axios.get(`/api/students/${props.id}`)
    setStudent(resp.data.data)
  }

  let [student, setStudent] = useState<Student>({ name: 'xx' })
  updateStudent()

  return <h3>姓名是：{student.name}</h3>
}
```

工作流程如下

首次使用 useState，用它的参数初始化 State

![image-20221005173958351](imgs/image-20221005173958351.png)

2s 后数据更新，setStudent 函数会更新 State 数据，并会触发下一次渲染（P5 的调用）

![image-20221005174347981](imgs/image-20221005174347981.png)

再次调用 useState，这时返回更新后的数据

![image-20221005174739593](imgs/image-20221005174739593.png)

这时再返回 jsx，内容就是 `姓名是：宋远桥` 了

> **P.S.**
>
> 使用了 useState 之后，会执行两次 xhr 请求，后一次请求是 react 开发工具发送的，不用理会



问题还没完：第二次调用 P5 时 updateStudent 还会执行，导致 2s 后响应返回又调用 setStudent，于是每隔 2s 就调用一次 P5（渲染一次）

![image-20221005175440228](imgs/image-20221005175440228.png)

怎么让 updateStudent 只执行一次？土办法是再设一个布尔 State 

![image-20221005181042078](imgs/image-20221005181042078.png)

接下来数据更新

![image-20221005181428984](imgs/image-20221005181428984.png)

第二次进入 P5 函数时，因为 fetch 条件不成立，就不会再执行两个 setXXX 了

![image-20221005182505908](imgs/image-20221005182505908.png)



函数式组件的工作流程

* 首次调用函数组件，返回的 jsx 代码会被渲染成【虚拟 dom 节点】（也称 Fiber 节点）
  * 此时使用 useState 会将组件工作过程中需要数据绑定到【虚拟 dom 节点】上
  * 根据【虚拟 dom 节点】会生成【真实 dom 节点】，由浏览器显示出来
* 当函数组件的 props 或 state 发生变化时，才会重新调用函数组件，返回 jsx
  * props 变化由父组件决定，state 变化由组件自身决定
  * jsx 与上次的【虚拟 dom 节点】对比
    * 如果没变化，复用上次的节点
    * 有变化，创建新的【虚拟 dom 节点】替换掉上次的节点



#### useEffect

Effect 叫副作用（不含贬义）。函数组件的主要目的是渲染生成 html，除了这个主功能之外，管理状态、fetch 数据等等，都可以算副作用。

useXXX 开头的一系列方法都是为副作用而生的，react 里叫它们 Hooks

useEffect 三种用法

```tsx
import axios from "axios"
import { useEffect, useState } from "react"

/*
useEffect
  参数1：箭头函数, 在真正渲染 html 之前会执行它
  参数2：
    情况1：没有, 代表每次执行组件函数时, 都会执行副作用函数
    情况2：[], 代表副作用函数只会执行一次
    情况3：[依赖项], 依赖项变化时，副作用函数会执行
*/
export default function P6({ id, age }: { id: number, age: number }) {

  console.log('1.主要功能')
    
  // useEffect(() => console.log('3.副作用功能')) 
  // useEffect(() => console.log('3.副作用功能'), []) 
  useEffect(() => console.log('3.副作用功能'), [id]) 

  console.log('2.主要功能')

  return <h3>{id}</h3>
}
```

用它改写 P5 案例

```tsx
import axios from "axios"
import { useEffect, useState } from "react"

export default function P6({ id, age }: { id: number, age: number }) {

  const [student, setStudent] = useState({name:'xx'})

  useEffect(()=>{
    async function updateStudent() {
      const resp = await axios.get(`http://localhost:8080/api/students/${id}`)    
      setStudent(resp.data.data)
    }
    updateStudent()
  }, [id])

  return <h3>{student.name}</h3>
}
```



#### useContext

```tsx
import axios from 'axios'
import { createContext, useContext, useEffect, useState } from 'react'
import { R, Student } from '../model/Student'

/*
  createContext         创建上下文对象
  useContext            读取上下文对象的值
  <上下文对象.Provider>  修改上下文对象的值
*/
const HiddenContext = createContext(false)

// 给以下组件提供数据，控制年龄隐藏、显示
export default function P7() {
  const [students, setStudents] = useState<Student[]>([])
  const [hidden, setHidden] = useState(false)
  useEffect(()=>{
    async function updateStudents() {
      const resp = await axios.get<R<Student[]>>("http://localhost:8080/api/students")
      setStudents(resp.data.data)
    }
    updateStudents()
  }, [])

  function hideOrShow() {
    // 参数：上一次状态值，旧值
    // 返回值：要更新的新值
    setHidden((old)=>{
      return !old
    })
  }
  return <HiddenContext.Provider value={hidden}>
    <input type="button" value={hidden?'显示':'隐藏'} onClick={hideOrShow}/>
    <P71 students={students}></P71>
  </HiddenContext.Provider>  
}

// 负责处理学生集合
function P71({ students }: { students: Student[] }) {
  const list = students.map(s=><P72 student={s} key={s.id}></P72>)
  return <>{list}</>
}

// 负责显示单个学生
function P72({ student }: { student: Student }) {
  const hidden = useContext(HiddenContext)
  const jsx = !hidden && <span>{student.age}</span>
  return <div>{student.name} {jsx}</div>
}
```

* 如果组件分散在多个文件中，HiddenContext 应该 export 导出，用到它的组件 import 导入
* React 中因修改触发的组件重新渲染，都应当是自上而下的
* setHidden 方法如果更新的是对象，那么要返回一个新对象，而不是在旧对象上做修改



#### 表单

```tsx
import axios from 'axios'
import React, { useState } from 'react'
import '../css/P8.css'

export default function P8() {

  const [student, setStudent] = useState({name:'', sex:'男', age:18})
  const [message, setMessage] = useState('')

  const options = ['男', '女']
  const jsx = options.map(e => <option key={e}>{e}</option>)

  // e 事件对象, e.target 事件源
  function onChange(e : React.ChangeEvent<HTMLInputElement|HTMLSelectElement>) {
    setStudent((old)=>{
      // 返回的新值，不能与旧值是同一个对象
      return {...old, [e.target.name]:e.target.value}
    })
  }

  async function onClick() {
    const resp = await axios.post('http://localhost:8080/api/students', student)
    setMessage(resp.data.data)
  }
  
  const messageJsx = message && <div className='success'>{message}</div>

  return (
    <form>
      <div>
        <label>姓名</label>
        <input type="text" value={student.name} onChange={onChange} name='name'/>
      </div>
      <div>
        <label>性别</label>
        <select value={student.sex} onChange={onChange} name='sex'>
          {jsx}
        </select>
      </div>
      <div>
        <label>年龄</label>
        <input type="text" value={student.age} onChange={onChange} name='age' />
      </div>
      <div>
        <input type='button' value='新增' onClick={onClick}/>
      </div>
      {messageJsx}
    </form>
  )
}
```



## 2. React 进阶

### 1) Ant Design

react 组件库

* 官网地址：https://ant.design/

* 镜像地址1：https://ant-design.gitee.io/

* 镜像地址2：https://ant-design.antgroup.com/

#### 入门

安装

```cmd
npm install antd
```

* 目前版本是 4.x

引入样式，在 css 文件中加入

```tsx
@import '~antd/dist/antd.css';
```

引入 antd 组件

```tsx
import { Button } from "antd";

export default function A1() {
  return <Button type='primary'>按钮</Button>
}
```



#### 国际化

试试其它组件

```tsx
import { Button, Modal } from "antd";

export default function A1() {
  return <Modal open={true} title='对话框'>内容</Modal>
}
```

发现确定和取消按钮是英文的，因为 antd 支持多语言，而默认语言是英文



想改成中文，建议修改最外层的组件 index.tsx

```tsx
// ...
import { ConfigProvider } from 'antd'
import zhCN from 'antd/es/locale/zh_CN'

root.render(
  <ConfigProvider locale={zhCN}>
    <A1></A1>
  </ConfigProvider>
)
```



#### 表格

```tsx
import { Table } from 'antd'
import { ColumnsType } from 'antd/lib/table'
import axios from 'axios'
import { useEffect, useState } from 'react'
import { R, Student } from '../model/Student'

export default function A3() {
  const [students, setStudents] = useState<Student[]>([])
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    async function getStudents() {
      const resp = await axios.get<R<Student[]>>(
        'http://localhost:8080/api/students'
      )
      setStudents(resp.data.data)
      setLoading(false)
    }

    getStudents()
  }, [])

  // title: 列标题  dataIndex: 要关联的属性名
  const columns: ColumnsType<Student> = [
    {
      title: '编号',
      dataIndex: 'id',
    },
    {
      title: '姓名',
      dataIndex: 'name',
    },
    {
      title: '性别',
      dataIndex: 'sex',
    },
    {
      title: '年龄',
      dataIndex: 'age',
    },
  ]

  // columns: 列定义
  // dataSource: 数据源，一般是数组包对象
  // rowKey: 作为唯一标识的属性名
  // loading: 显示加载图片
  return (
    <Table
      columns={columns}
      dataSource={students}
      rowKey='id'
      loading={loading}></Table>
  )
}
```



#### 客户端分页

```tsx
import { Table } from 'antd'
import { ColumnsType, TablePaginationConfig } from 'antd/lib/table'
import axios from 'axios'
import { useEffect, useState } from 'react'
import { R, Student } from '../model/Student'

export default function A3() {
  const [students, setStudents] = useState<Student[]>([])
  const [loading, setLoading] = useState(true)
  const [pagination, setPagination] = useState<TablePaginationConfig>(
    {current:1, pageSize:5}
  )

  // 参数: 新的分页数据
  function onTableChange(newPagination: TablePaginationConfig) {
    setPagination(newPagination)
  }

  useEffect(() => {
    async function getStudents() {
      const resp = await axios.get<R<Student[]>>(
        'http://localhost:8080/api/students'
      )
      setStudents(resp.data.data)
      setLoading(false)
    }

    getStudents()
  }, [])

  // ... 省略

  // pagination: 分页数据
  // onChange: 当页号，页大小改变时触发
  return (
    <Table
      columns={columns}
      dataSource={students}
      rowKey='id'
      loading={loading}
      pagination={pagination}
      onChange={onTableChange}></Table>
  )
}
```

* 本例还是查询所有数据，分页是客户端 Table 组件自己实现的



#### 服务端分页

```tsx
import { Table } from 'antd'
import { ColumnsType, TablePaginationConfig } from 'antd/lib/table'
import axios from 'axios'
import { useEffect, useState } from 'react'
import { PageResp, R, Student } from '../model/Student'

export default function A4() {
  const [students, setStudents] = useState<Student[]>([])
  const [loading, setLoading] = useState(true)
  const [pagination, setPagination] = useState<TablePaginationConfig>({
    current: 1,
    pageSize: 5,
  })

  function onTableChange(newPagination: TablePaginationConfig) {
    setPagination(newPagination)
  }

  useEffect(() => {
    async function getStudents() {
      // params 用来给请求添加 url 后的 ? 参数
      const resp = await axios.get<R<PageResp<Student>>>(
        'http://localhost:8080/api/students/q',
        {
          params: {
            page: pagination.current,
            size: pagination.pageSize,
          },
        }
      )
      // 返回结果中：list 代表当前页集合, total 代表总记录数
      setStudents(resp.data.data.list)
      setPagination((old) => {
        return { ...old, total: resp.data.data.total }
      })
      setLoading(false)
    }

    getStudents()
  }, [pagination.current, pagination.pageSize])
  // useEffect 需要在依赖项( current 和 pageSize ) 改变时重新执行

  const columns: ColumnsType<Student> = [
    {
      title: '编号',
      dataIndex: 'id',
    },
    {
      title: '姓名',
      dataIndex: 'name',
    },
    {
      title: '性别',
      dataIndex: 'sex',
    },
    {
      title: '年龄',
      dataIndex: 'age',
    },
  ]

  return (
    <Table
      columns={columns}
      dataSource={students}
      rowKey='id'
      loading={loading}
      pagination={pagination}
      onChange={onTableChange}></Table>
  )
}
```

* 本例需要服务端配合来实现分页，参见代码中新加的注释

其中 PageResp 类型定义为

```tsx
export interface PageResp<T> {
  list: T[],
  total: number
}
```



#### 条件查询

```tsx
import { Input, Select, Table } from 'antd'
import { ColumnsType, TablePaginationConfig } from 'antd/lib/table'
import axios from 'axios'
import React, { useEffect, useState } from 'react'
import { PageResp, R, Student, StudentQueryForm } from '../model/Student'

const { Option } = Select

export default function A5() {
  const [students, setStudents] = useState<Student[]>([])
  const [loading, setLoading] = useState(true)
  const [pagination, setPagination] = useState<TablePaginationConfig>({
    current: 1,
    pageSize: 5,
  })
  // 代表查询条件的状态数据
  const [form, setForm] = useState<StudentQueryForm>({})

  function onTableChange(newPagination: TablePaginationConfig) {
    setPagination(newPagination)
  }

  useEffect(() => {
    async function getStudents() {
      const resp = await axios.get<R<PageResp<Student>>>(
        'http://localhost:8080/api/students/q',
        {
          params: {
            page: pagination.current,
            size: pagination.pageSize,
            ...form // 补充查询参数
          },
        }
      )
      setStudents(resp.data.data.list)
      setPagination((old) => {
        return { ...old, total: resp.data.data.total }
      })
      setLoading(false)
    }

    getStudents()
  }, [pagination.current, pagination.pageSize, form.name, form.sex, form.age])
  // 依赖项除了分页条件外，新加了查询条件依赖
    
  const columns: ColumnsType<Student> = [
    {
      title: '编号',
      dataIndex: 'id',
    },
    {
      title: '姓名',
      dataIndex: 'name',
    },
    {
      title: '性别',
      dataIndex: 'sex',
    },
    {
      title: '年龄',
      dataIndex: 'age',
    },
  ]

  // name 条件改变时处理函数
  function onNameChange(e: React.ChangeEvent<HTMLInputElement>) {
    setForm((old)=>{
      return {...old, name: e.target.value}
    })
  }

  // sex 条件改变时处理函数
  function onSexChange(value: string) {
    setForm((old)=>{
      return {...old, sex: value}
    })
  }

  // age 条件改变时处理函数
  function onAgeChange(value: string) {
    setForm((old)=>{
      return {...old, age: value}
    })
  }

  return (
    <div>
      <div>
        <Input
          style={{ width: 120 }}
          placeholder='请输入姓名'
          value={form.name}
          onChange={onNameChange}></Input>
        <Select
          style={{ width: 120 }}
          placeholder='请选择性别'
          allowClear={true}
          value={form.sex}
          onChange={onSexChange}>
          <Option value='男'>男</Option>
          <Option value='女'>女</Option>
        </Select>
        <Select
          style={{ width: 120 }}
          placeholder='请选择年龄'
          allowClear={true}
          value={form.age}
          onChange={onAgeChange}>
          <Option value='1,19'>20以下</Option>
          <Option value='20,29'>20左右</Option>
          <Option value='30,39'>30左右</Option>
          <Option value='40,120'>40以上</Option>
        </Select>
      </div>
      <Table
        columns={columns}
        dataSource={students}
        rowKey='id'
        loading={loading}
        pagination={pagination}
        onChange={onTableChange}></Table>
    </div>
  )
}
```

* 建议 axios 发请求时用 params 而不要自己拼字符串，因为自己拼串需要去掉值为 undefined 的属性

其中 StudentQueryForm 为

```tsx
export interface StudentQueryForm {
  name?: string,
  sex?: string,
  age?: string,
  [key: string]: any
}
```



#### 删除

```tsx
import { Button, message, Popconfirm } from 'antd'
import axios from 'axios'
import { R } from '../model/Student'

export default function A6Delete({ id, onSuccess }: { id: number, onSuccess:()=>void }) {
  async function onConfirm() {
    const resp = await axios.delete<R<string>>(
      `http://localhost:8080/api/students/${id}`
    )
    message.success(resp.data.data)
    // 改变 form 依赖项
    onSuccess()
  }
  return (
    <Popconfirm title='确定要删除学生吗?' onConfirm={onConfirm}>
      <Button danger size='small'>
        删除
      </Button>
    </Popconfirm>
  )
}
```

使用删除组件

```tsx
import { Button, Input, Select, Space, Table } from 'antd'
import { ColumnsType, TablePaginationConfig } from 'antd/lib/table'
import axios from 'axios'
import React, { useEffect, useState } from 'react'
import { PageResp, R, Student, StudentQueryForm } from '../model/Student'
import A6Delete from './A6Delete'

const { Option } = Select

export default function A6() {
  // ... 省略

  function onDeleteSuccess() {
    setForm((old)=>{
      return {...old}
    })
  }
    
  const columns: ColumnsType<Student> = [
    // ... 省略
    {
      title: '操作',
      dataIndex: 'operation',
      // value: 属性值, student
      render: (_, student)=>{
        return <>
          <Space>
            <A6Delete id={student.id} onSuccess={onDeleteSuccess}></A6Delete>
            <Button type='default' size='small'>修改</Button>
          </Space>
        </>
      }
    }
  ]

  // ... 省略
}
```



#### 修改

```tsx
import { Form, Input, InputNumber, message, Modal, Radio } from 'antd'
import { Rule } from 'antd/lib/form'
import axios from 'axios'
import { useEffect } from 'react'
import { R, Student } from '../model/Student'

export default function A6Update({
  open,
  student,
  onSuccess,
  onCancel,
}: {
  open: boolean
  student: Student
  onSuccess?: () => void
  onCancel?: () => void
}) {
  const { Item } = Form
  const { Group } = Radio
  const options = [
    { label: '男', value: '男' },
    { label: '女', value: '女' },
  ]

  const [form] = Form.useForm() // 代表了表单对象

  const nameRules: Rule[] = [
    { required: true, message: '姓名必须' },
    { min: 2, type: 'string', message: '至少两个字符' },
  ]

  const ageRules: Rule[] = [
    { required: true, message: '年龄必须' },
    { min: 1, type: 'number', message: '最小1岁' },
    { max: 120, type: 'number', message: '最大120岁' },
  ]

  async function onOk() {
    // 验证并获取表单数据
    try {
      const values = await form.validateFields()
      console.log(values)
      const resp = await axios.put<R<string>>(
        `http://localhost:8080/api/students/${values.id}`,
        values
      )
      message.success(resp.data.data)
      onSuccess && onSuccess()
    } catch (e) {
      console.error(e)
    }
  }

  useEffect(() => {
    // 修改表单数据
    form.setFieldsValue(student) // id, name, sex, age
  }, [student])

  return (
    <Modal
      open={open}
      title='修改学生'
      onOk={onOk}
      onCancel={onCancel}
      forceRender={true}>
      <Form form={form}>
        <Item label='编号' name='id'>
          <Input readOnly></Input>
        </Item>
        <Item label='姓名' name='name' rules={nameRules}>
          <Input></Input>
        </Item>
        <Item label='性别' name='sex'>
          <Group
            options={options}
            optionType='button'
            buttonStyle='solid'></Group>
        </Item>
        <Item label='年龄' name='age' rules={ageRules}>
          <InputNumber></InputNumber>
        </Item>
      </Form>
    </Modal>
  )
}
```

* forceRender 是避免因为使用 useForm 后，表单套在 Modal 中会出现的警告错误

使用组件

```tsx
import { Button, Input, Select, Space, Table } from 'antd'
import { ColumnsType, TablePaginationConfig } from 'antd/lib/table'
import axios from 'axios'
import React, { useEffect, useState } from 'react'
import { PageResp, R, Student, StudentQueryForm } from '../model/Student'
import A6Delete from './A6Delete'
import A6Update from './A6Update'

const { Option } = Select

export default function A6() {
  // ... 省略
  const columns: ColumnsType<Student> = [
    // ... 省略
    {
      title: '操作',
      dataIndex: 'operation',
      // value: 属性值, student
      render: (_, student) => {
        return (
          <>
            <Space>
              <A6Delete id={student.id} onSuccess={onDeleteSuccess}></A6Delete>
              <Button
                type='default'
                size='small'
                onClick={() => {
                  onUpdateClick(student)
                }}>
                修改
              </Button>
            </Space>
          </>
        )
      },
    },
  ]

  // -------------- 修改功能开始 -------------
  function onUpdateClick(student: Student) {
    setUpdateOpen(true)
    setUpdateForm(student)
  }

  function onUpdateCancel() {
    setUpdateOpen(false)
  }

  function onUpdateSuccess() {
    setUpdateOpen(false)
    setForm((old) => {
      return { ...old }
    })
  }

  const [updateOpen, setUpdateOpen] = useState(false)
  const [updateForm, setUpdateForm] = useState<Student>({
    id: 0,
    name: '',
    sex: '男',
    age: 18,
  })
  // -------------- 修改功能结束 -------------

  return (
    <div>
      <A6Update
        open={updateOpen}
        student={updateForm}
        onSuccess={onUpdateSuccess}
        onCancel={onUpdateCancel}></A6Update>
      <!-- ... 省略 -->
    </div>
  )
}
```



#### 新增

```tsx
import { Form, Input, InputNumber, message, Modal, Radio } from 'antd'
import { Rule } from 'antd/lib/form'
import axios from 'axios'
import { useEffect } from 'react'
import { R, Student } from '../model/Student'

export default function A6Insert({
  open,
  student,
  onSuccess,
  onCancel,
}: {
  open: boolean
  student: Student
  onSuccess?: () => void
  onCancel?: () => void
}) {
  const { Item } = Form
  const { Group } = Radio
  const options = [
    { label: '男', value: '男' },
    { label: '女', value: '女' },
  ]

  const [form] = Form.useForm() // 代表了表单对象

  const nameRules: Rule[] = [
    { required: true, message: '姓名必须' },
    { min: 2, type: 'string', message: '至少两个字符' },
  ]

  const ageRules: Rule[] = [
    { required: true, message: '年龄必须' },
    { min: 1, type: 'number', message: '最小1岁' },
    { max: 120, type: 'number', message: '最大120岁' },
  ]

  async function onOk() {
    // 验证并获取表单数据
    try {
      const values = await form.validateFields()
      console.log(values)
      const resp = await axios.post<R<string>>(
        `http://localhost:8080/api/students`,
        values
      )
      message.success(resp.data.data)
      onSuccess && onSuccess()
      form.resetFields() // 重置表单
    } catch (e) {
      console.error(e)
    }
  }

  return (
    <Modal
      open={open}
      title='新增学生'
      onOk={onOk}
      onCancel={onCancel}
      forceRender={true}>
      <Form form={form} initialValues={student}>
        <Item label='姓名' name='name' rules={nameRules}>
          <Input></Input>
        </Item>
        <Item label='性别' name='sex'>
          <Group
            options={options}
            optionType='button'
            buttonStyle='solid'></Group>
        </Item>
        <Item label='年龄' name='age' rules={ageRules}>
          <InputNumber></InputNumber>
        </Item>
      </Form>
    </Modal>
  )
}

```

* initialValues 只会触发一次表单赋初值
* form.resetFields() 会将表单重置为 initialValues 时的状态

使用组件

```tsx
import { Button, Input, Select, Space, Table } from 'antd'
import { ColumnsType, TablePaginationConfig } from 'antd/lib/table'
import axios from 'axios'
import React, { useEffect, useState } from 'react'
import { PageResp, R, Student, StudentQueryForm } from '../model/Student'
import A6Delete from './A6Delete'
import A6Insert from './A6Insert'
import A6SelectedDelete from './A6SelectedDelete'
import A6Update from './A6Update'

const { Option } = Select

export default function A6() {
  // ... 省略

  // -------------- 新增功能开始 -------------
  function onInsertClick() {
    setInsertOpen(true)
  }

  function onInsertCancel() {
    setInsertOpen(false)
  }

  function onInsertSuccess() {
    setInsertOpen(false)
    setForm((old) => {
      return { ...old }
    })
  }

  const [insertOpen, setInsertOpen] = useState(false)
  const [insertForm, setInsertForm] = useState<Student>({
    id: 0,
    name: '',
    sex: '男',
    age: 18,
  })
  // -------------- 新增功能结束 -------------

  
  return (
    <div>
      <A6Insert
        open={insertOpen}
        student={insertForm}
        onSuccess={onInsertSuccess}
        onCancel={onInsertCancel}></A6Insert>
      <A6Update
        open={updateOpen}
        student={updateForm}
        onSuccess={onUpdateSuccess}
        onCancel={onUpdateCancel}></A6Update>
      <div>
        <Space>
          <Input
            style={{ width: 120 }}
            placeholder='请输入姓名'
            value={form.name}
            onChange={onNameChange}></Input>
          <Select
            style={{ width: 120 }}
            placeholder='请选择性别'
            allowClear={true}
            value={form.sex}
            onChange={onSexChange}>
            <Option value='男'>男</Option>
            <Option value='女'>女</Option>
          </Select>
          <Select
            style={{ width: 120 }}
            placeholder='请选择年龄'
            allowClear={true}
            value={form.age}
            onChange={onAgeChange}>
            <Option value='1,19'>20以下</Option>
            <Option value='20,29'>20左右</Option>
            <Option value='30,39'>30左右</Option>
            <Option value='40,120'>40以上</Option>
          </Select>

          <Button type='primary' onClick={onInsertClick}>新增</Button>
        </Space>
      </div>
      <Table
        columns={columns}
        dataSource={students}
        rowKey='id'
        loading={loading}
        pagination={pagination}
        onChange={onTableChange}></Table>
    </div>
  )
}

```



#### 删除选中

```tsx
import { Button, message, Popconfirm } from "antd";
import axios from "axios";
import React from "react";
import { R } from "../model/Student";

export default function A6DeleteSelected(
  {ids, onSuccess}: {ids:React.Key[], onSuccess?:()=>void} // Key[] 是 number 或 string 的数组
){
  const disabled = ids.length === 0
  async function onConfirm() {
    const resp = await axios.delete<R<string>>('http://localhost:8080/api/students', {
      data: ids
    })
    message.success(resp.data.data)
    onSuccess && onSuccess()
  }
  return (
    <Popconfirm title='真的要删除选中的学生吗?' onConfirm={onConfirm} disabled={disabled}>
      <Button danger type='primary' disabled={disabled}>
        删除选中
      </Button>
    </Popconfirm>
  )
}
```

与 A6 结合

```tsx
import { Button, Input, Select, Space, Table } from 'antd'
import { ColumnsType, TablePaginationConfig } from 'antd/lib/table'
import axios from 'axios'
import React, { useEffect, useState } from 'react'
import { PageResp, R, Student, StudentQueryForm } from '../model/Student'
import A6Delete from './A6Delete'
import A6Insert from './A6Insert'
import A6SelectedDelete from './A6SelectedDelete'
import A6Update from './A6Update'

const { Option } = Select

export default function A6() {
  // ... 省略

  // -------------- 删除选中功能开始 -------------
  const [ids, setIds] = useState<React.Key[]>([])
  function onIdsChange(ids:React.Key[]) {
    // console.log(ids)
    setIds(ids)
  }
  function onDeleteSelectedSuccess() {
    setForm((old)=>{
      return {...old}
    })
    setIds([])
  }
  // -------------- 删除选中功能结束 -------------
  return (
    <div>
      <A6Insert
        open={insertOpen}
        student={insertForm}
        onSuccess={onInsertSuccess}
        onCancel={onInsertCancel}></A6Insert>
      <A6Update
        open={updateOpen}
        student={updateForm}
        onSuccess={onUpdateSuccess}
        onCancel={onUpdateCancel}></A6Update>
      <div>
        <Space>
          <!-- ... 省略 -->
          <A6SelectedDelete ids={ids} onSuccess={onDeleteSelectedSuccess}></A6SelectedDelete>
        </Space>
      </div>
      <Table
        rowSelection={{
          selectedRowKeys: selectedKeys,
          onChange: onSelectChange,
        }}
        columns={columns}
        dataSource={students}
        rowKey='id'
        loading={loading}
        pagination={pagination}
        onChange={onTableChange}></Table>
    </div>
  )
}
```



#### useRequest

安装

```cmd
npm install ahooks
```

使用

```tsx
import { useRequest } from 'ahooks'
import { Table } from 'antd'
import { ColumnsType } from 'antd/lib/table'
import axios from 'axios'
import { Student, R } from '../model/Student'

export default function A3() {
  function getStudents() {
    return axios.get<R<Student[]>>('http://localhost:8080/api/students')
  }

  const { loading, data } = useRequest(getStudents)  

  const columns: ColumnsType<Student> = [
    {
      title: '编号',
      dataIndex: 'id',
    },
    {
      title: '姓名',
      dataIndex: 'name',
    },
    {
      title: '性别',
      dataIndex: 'sex',
    },
    {
      title: '年龄',
      dataIndex: 'age',
    },
  ]

  return (
    <Table
      dataSource={data?.data.data}
      columns={columns}
      rowKey='id'
      loading={loading}
      pagination={{ hideOnSinglePage: true }}></Table>
  )
}
```



#### useAntdTable

```tsx
import { useAntdTable } from 'ahooks'
import { Table } from 'antd'
import { ColumnsType } from 'antd/lib/table'
import axios from 'axios'
import { Student, R } from '../model/Student'

interface PageResp<T> {
  total: number
  list: T[]
}

interface PageReq {
  current: number
  pageSize: number
  sorter?: any
  filter?: any
}

export default function A3() {
  async function getStudents({ current, pageSize }: PageReq) {
    const resp = await axios.get<R<PageResp<Student>>>(
      `http://localhost:8080/api/students/q?page=${current}&size=${pageSize}`
    )
    return resp.data.data
  }

  const { tableProps } = useAntdTable(getStudents, {
    defaultParams: [{ current: 1, pageSize: 5 }],
  })
  console.log(tableProps)

  const columns: ColumnsType<Student> = [
    {
      title: '编号',
      dataIndex: 'id',
    },
    {
      title: '姓名',
      dataIndex: 'name',
    },
    {
      title: '性别',
      dataIndex: 'sex',
    },
    {
      title: '年龄',
      dataIndex: 'age',
    },
  ]

  return <Table {...tableProps} columns={columns} rowKey='id'></Table>
}
```



### 2) MobX

#### 介绍

场景：组件0 改了数据，其它组件也想拿到改后的数据，如图所示

![image-20221025104453534](imgs/image-20221025104453534.png)

这种多个组件间要共享状态数据的情况，useState 不够用，useContext 也不好用了



能和 react 配合使用的状态管理库有

* MobX
* Redux

其中 Redux 的 API **非常**难用，所以我选了更符合**人类习惯**的 MobX；它虽然是面向对象语法，但也能和函数式代码很好地结合





#### 文档

* [MobX 中文文档](https://cn.mobx.js.org/)
* [MobX 官方文档](https://mobx.js.org/README.html)



#### 安装

```
npm install mobx mobx-react-lite
```

* mobx 目前版本是 6.x
* mobx-react-lite 目前版本是 3.x



#### 名词

![Action, State, View](https://mobx.js.org/assets/action-state-view.png)

* Actions 用来修改状态数据的方法
* Observable state 状态数据，可观察
* Derived values 派生值，也叫 Computed values 计算值，会根据状态数据的改变而改变，具有缓存功能
* Reactions 状态数据发生变化后要执行的操作，如 react 函数组件被重新渲染



#### 使用

先定义一个在函数外存储状态数据的 store，它和 useState 不同：

* useState 里的状态数据是存储在每个组件节点上，不同组件之间没法共享
* 而 MobX 的 store 就是一个普通 js 对象，只要保证多个组件都访问此对象即可

```tsx
import axios from 'axios'
import { makeAutoObservable, runInAction } from 'mobx'
import { R, Student } from '../model/Student'

class StudentStore {
  student: Student = { id: 0, name: '', sex: '男', age: 18, photo: '' }

  constructor() {
    makeAutoObservable(this)
  }

  async fetch(id: number) {
    const resp = await axios.get<R<Student>>(
      `http://localhost:8080/api/students/${id}`
    )
    runInAction(() => {
      this.student = resp.data.data
    })
  }
    
  get print() {
    const first = this.student.name.charAt(0)
    if (this.student.sex === '男') {
      return first.concat('大侠')
    } else if (this.student.sex === '女') {
      return first.concat('女侠')
    } else {
      return ''
    }
  } 
}

export default new StudentStore()
```

其中 makeAutoObservable 会

* 将对象的属性 student 变成 Observable state，即状态数据
* 将对象的方法 fetch 变成 Action，即修改数据的方法
* 将 get 方法变成 Computed values

在异步操作里为状态属性赋值，需要放在 runInAction 里，否则会有警告错误



使用 store 时，所有用到它的组件都要用 observer 包装，才能感知状态变化，对应图中的 reactions

```tsx
import Search from 'antd/lib/input/Search'
import { observer } from 'mobx-react-lite'
import studentStore from '../store/StudentStore'
import A71 from './A71'
import A72 from './A72'

const A7 = () => {
  return (
    <div>
      <Search
        placeholder='input search text'
        onSearch={(v) => studentStore.fetch(Number(v))}
        style={{ width: 100 }}
      />
      <h3>组件0 {studentStore.student.name}</h3>
      <A71></A71>
      <A72></A72>
    </div>
  )
}

export default observer(A7)
```

其它组件

```tsx
import { observer } from 'mobx-react-lite'
import studentStore from '../store/StudentStore'

const A71 = () =>{
  return <h3 style={{color:'red'}}>组件1 {studentStore.student.name}</h3>
}

export default observer(A71)
```



```tsx
import { observer } from 'mobx-react-lite'
import studentStore from '../store/StudentStore'

const A72 = () =>{
  return <h3 style={{color:'red'}}>组件2 {studentStore.student.name}</h3>
}

export default observer(A72)
```



#### 注解方式

```typescript
import { R, Student } from "../model/Student";
import { action, computed, makeAutoObservable, makeObservable, observable, runInAction } from 'mobx'
import axios from "axios";

class StudentStore {
  // 属性 - 对应状态数据 observable state
  @observable student: Student = { id: 0, name: '', sex: '男', age: 18, photo: '' }
  // 方法 - 对应 action 方法
  @action setName(name: string) {
    this.student.name = name
  }
  @action async fetch(id: number) {
    const resp = await axios.get<R<Student>>(`http://localhost:8080/api/students/${id}`)
    runInAction(() => {
      this.student = resp.data.data
    })
  }
  // get 方法 - 对应 derived value
  @computed get displayName() {
    const first = this.student.name.charAt(0)
    if (this.student.sex === '男') {
      return first + '大侠'
    } else if (this.student.sex === '女') {
      return first + '女侠'
    } else {
      return ''
    }
  }
  // 构造器
  constructor() {
    makeObservable(this)
  }
}

export default new StudentStore()
```

需要在 tsconfig.json 中加入配置

```json
{
  "compilerOptions": {
    // ...
    "experimentalDecorators": true
  }
}
```



### 3) React Router

#### 安装

```cmd
npm install react-router-dom
```

* 目前版本是 6.x



#### 使用

新建文件 src/router/router.tsx

```tsx
import { lazy } from 'react'
import { Navigate, RouteObject, useRoutes } from 'react-router-dom'

export function load(name: string) {
  const Page = lazy(() => import(`../pages/${name}`))
  return <Page></Page>
}

const staticRoutes: RouteObject[] = [
  { path: '/login', element: load('A8Login') },
  {
    path: '/',
    element: load('A8Main'),
    children: [
      { path: 'student', element: load('A8MainStudent') },
      { path: 'teacher', element: load('A8MainTeacher') },
      { path: 'user', element: load('A8MainUser') }
    ],
  },
  { path: '/404', element: load('A8Notfound') },
  { path: '/*', element: <Navigate to={'/404'}></Navigate> },
]

export default function Router() {
  return useRoutes(staticRoutes)
}
```

index.tsx 改成

```tsx
import ReactDOM from 'react-dom/client';
import './index.css';
import { ConfigProvider } from 'antd';
import zhCN from 'antd/es/locale/zh_CN'

import { BrowserRouter } from 'react-router-dom';
import Router from './router/router';

const root = ReactDOM.createRoot(
  document.getElementById('root') as HTMLElement
);

root.render(
  <ConfigProvider locale={zhCN}>
    <BrowserRouter>
      <Router></Router>
    </BrowserRouter>
  </ConfigProvider>  
)
```

A8Main 的代码

```tsx
import { Layout } from "antd";
import { Link, Outlet } from "react-router-dom";

export default function A8Main () {  
  return <Layout>
    <Layout.Header>头部导航</Layout.Header>
    <Layout>
      <Layout.Sider>侧边导航
        <Link to='/student'>学生管理</Link>
        <Link to='/teacher'>教师管理</Link>
        <Link to='/user'>用户管理</Link>
      </Layout.Sider>
      <Layout.Content>
        <Outlet></Outlet>
      </Layout.Content>
    </Layout>
  </Layout>
}
```

1. Navigate 的作用是重定向
2. load 方法的作用是懒加载组件，更重要的是根据字符串找到真正的组件，这是动态路由所需要的
3. children 来进行嵌套路由映射，嵌套路由在跳转后，并不是替换整个页面，而是用新页面替换父页面的 Outlet 部分



#### 动态路由

路由分成两部分：

* 静态路由，固定的部分，如主页、404、login 这几个页面
* 动态路由，变化的部分，经常是主页内的嵌套路由，比如 Student、Teacher 这些

动态路由应该在用户登录后、根据角色不同从后端获取；因为数据会变，所以用 mobx 管理

```tsx
import axios from 'axios'
import { makeAutoObservable, runInAction } from 'mobx'
import { Navigate, RouteObject } from 'react-router-dom'
import { MenuAndRoute, R, Route } from '../model/Student'
import { load } from '../router/MyRouter'

class RoutesStore {
  dynamicRoutes: Route[]

  async fetch(username: string) {
    const resp = await axios.get<R<MenuAndRoute>>(
      `http://localhost:8080/api/menu/${username}`
    )
    runInAction(() => {
      this.dynamicRoutes = resp.data.data.routeList
      localStorage.setItem('dynamicRoutes', JSON.stringify(this.dynamicRoutes))
    })
  }

  constructor() {
    makeAutoObservable(this)
    const r = localStorage.getItem('dynamicRoutes')
    this.dynamicRoutes = r ? JSON.parse(r) : []
  }

  reset() {
    this.dynamicRoutes = []
    localStorage.removeItem('dynamicRoutes')
  }

  get routes() {
    const staticRoutes: RouteObject[] = [
      { path: '/login', element: load('A8Login') },
      { path: '/', element: load('A8Main') },
      { path: '/404', element: load('A8Notfound') },
      { path: '/*', element: <Navigate to={'/404'}></Navigate> },
    ]
    const main = staticRoutes[1]

    main.children = this.dynamicRoutes.map((r) => {
      console.log(r.path, r.element)
      return {
        path: r.path,
        element: load(r.element),
      }
    })
    return staticRoutes
  }
}

export default new RoutesStore()
```

* 其中用 localStorage 进行了数据的持久化，避免刷新后丢失数据

MyRouter 文件改成

```tsx
import { observer } from 'mobx-react-lite'
import { lazy } from 'react'
import { Navigate, RouteObject, useRoutes } from 'react-router-dom'
import RoutesStore from '../store/RoutesStore'

// 把字符串组件 => 组件标签
export function load(name: string) {
  // A8Login
  const Page = lazy(() => import(`../pages/${name}`))
  return <Page></Page>
}

// 路由对象
function MyRouter() {  
  const router = useRoutes(RoutesStore.routes)
  return router
}

export default observer(MyRouter)
```

注意导入 router 对象时用 observer 做了包装，这样 store 变化时能重建 router 对象



#### 动态菜单

图标要独立安装依赖

```
npm install @ant-design/icons
```

图标组件，用来将**字符串图标**转换为**标签图标**

```tsx
import * as icons from '@ant-design/icons'

interface Module {
  [p: string]: any
}

const all: Module = icons

export default function Icon({ name }: { name: string }) {
  const Icon = all[name]
  return <Icon></Icon>
}
```

修改 RoutesStore.tsx

```tsx
import axios from 'axios'
import { makeAutoObservable, runInAction } from 'mobx'
import { Link, Navigate, RouteObject } from 'react-router-dom'
import { Menu, MenuAndRoute, R, Route } from '../model/Student'
import { load } from '../router/MyRouter'
import Icon from './Icon'

function convertMenu(m: Menu): any {
  const Label = m.routePath ? <Link to={m.routePath}>{m.label}</Link> : m.label
  return {
    label: Label,
    key: m.key,
    icon: <Icon name={m.icon}></Icon>,
    children: m.children && m.children.map(convertMenu)
  }
}

class RoutesStore {
  // 动态部分
  dynamicRoutes: Route[] = []
  dynamicMenus: Menu[] = []

  async fetch(username: string) {
    const resp = await axios.get<R<MenuAndRoute>>(
      `http://localhost:8080/api/menu/${username}`
    )
    runInAction(() => {
      this.dynamicRoutes = resp.data.data.routeList
      localStorage.setItem('dynamicRoutes', JSON.stringify(this.dynamicRoutes))

      this.dynamicMenus = resp.data.data.menuTree
      localStorage.setItem('dynamicMenus', JSON.stringify(this.dynamicMenus))
    })
  }

  get menus() {
    return this.dynamicMenus.map(convertMenu)
  }

  get routes() {
    const staticRoutes: RouteObject[] = [
      { path: '/login', element: load('A8Login') },
      { path: '/', element: load('A8Main'), children: [] },
      { path: '/404', element: load('A8Notfound') },
      { path: '/*', element: <Navigate to={'/404'}></Navigate> },
    ]
    staticRoutes[1].children = this.dynamicRoutes.map((r) => {
      return {
        path: r.path,
        element: load(r.element),
      }
    })
    return staticRoutes
  }

  constructor() {
    makeAutoObservable(this)
    const json = localStorage.getItem('dynamicRoutes')
    this.dynamicRoutes = json ? JSON.parse(json) : []

    const json2 = localStorage.getItem('dynamicMenus')
    this.dynamicMenus = json2 ? JSON.parse(json2) : []
  }

  reset() {
    localStorage.removeItem('dynamicRoutes')
    this.dynamicRoutes = []
    localStorage.removeItem('dynamicMenus')
    this.dynamicMenus = []
  }
}

export default new RoutesStore()

```

其中 convertMenu 是核心方法，负责把服务器返回的 Menu 转成 antd Menu 组件需要的 Menu

使用

```tsx
<Menu items={RoutesStore.menus} mode='inline' theme="dark"></Menu>
```

跳转若报错，可能是组件懒加载引起的，需要用 Suspense 解决

```tsx
import { Suspense } from 'react'

root.render(
  <ConfigProvider locale={zhCN}>
    <BrowserRouter>
      <Suspense fallback={<h3>加载中...</h3>}>
        <MyRouter></MyRouter>
      </Suspense>
    </BrowserRouter>
  </ConfigProvider>
)
```





#### 登录

```tsx
import { ItemType } from 'antd/lib/menu/hooks/useItems'
import axios from 'axios'
import { makeAutoObservable, runInAction } from 'mobx'
import { Link, Navigate, RouteObject } from 'react-router-dom'
import { LoginReq, LoginResp, Menu, MenuAndRoute, R, Route } from '../model/Student'
import { load } from '../router/MyRouter'
import Icon from './Icon'

function convertMenu(m: Menu): ItemType {
  const Label = m.routePath? <Link to={m.routePath}>{m.label}</Link> : m.label
  return {
    key: m.key,
    label: Label,
    icon: <Icon name={m.icon}></Icon>, 
    children: m.children && m.children.map(convertMenu)
  }
}

class RoutesStore {
  // 动态部分
  dynamicRoutes: Route[] = []
  dynamicMenus: Menu[] = []

  token: string = ''
  state: string = 'pending' // 取值 pending done error
  message: string = '' // 取值: 1. 空串 正常  2. 非空串 错误消息

  async login(loginReq: LoginReq) {
    this.state = 'pending'
    this.message = ''
    const resp1 = await axios.post<R<LoginResp>>(
      'http://localhost:8080/api/loginJwt',
      loginReq
    )
    if(resp1.data.code === 999) {
      const resp2 = await axios.get<R<MenuAndRoute>>(
        `http://localhost:8080/api/menu/${loginReq.username}`
      )
      runInAction(()=>{
        this.token = resp1.data.data.token
        localStorage.setItem('token', this.token)

        this.dynamicRoutes = resp2.data.data.routeList
        localStorage.setItem('dynamicRoutes', JSON.stringify(this.dynamicRoutes))

        this.dynamicMenus = resp2.data.data.menuTree
        localStorage.setItem('dynamicMenus', JSON.stringify(this.dynamicMenus))

        this.state = 'done'
      })
    } else {
      runInAction(()=>{
        this.message = resp1.data.message || '未知错误'
        this.state = 'error'
      })
    }
  }

  async fetch(username: string) {
    const resp = await axios.get<R<MenuAndRoute>>(
      `http://localhost:8080/api/menu/${username}`
    )
    runInAction(() => {
      this.dynamicRoutes = resp.data.data.routeList
      localStorage.setItem('dynamicRoutes', JSON.stringify(this.dynamicRoutes))

      this.dynamicMenus = resp.data.data.menuTree
      localStorage.setItem('dynamicMenus', JSON.stringify(this.dynamicMenus))
    })
  }
    
  get routes() {
    const staticRoutes: RouteObject[] = [
      { path: '/login', element: load('A8Login') },
      { path: '/', element: load('A8Main'), children: [] },
      { path: '/404', element: load('A8Notfound') },
      { path: '/*', element: <Navigate to={'/404'}></Navigate> },
    ]
    staticRoutes[1].children = this.dynamicRoutes.map((r) => {
      return {
        path: r.path,
        element: load(r.element),
      }
    })
    return staticRoutes
  }

  get menus() {
    return this.dynamicMenus.map(convertMenu)
  }

  constructor() {
    makeAutoObservable(this)
    const json = localStorage.getItem('dynamicRoutes')
    this.dynamicRoutes = json ? JSON.parse(json) : []

    const json1 = localStorage.getItem('dynamicMenus')
    this.dynamicMenus = json1 ? JSON.parse(json1) : []

    const token = localStorage.getItem('token')
    this.token = token ?? ''
      
    this.message = ''
    this.state = 'pending'  
  }

  reset() {
    localStorage.removeItem('dynamicRoutes')
    this.dynamicRoutes = []

    localStorage.removeItem('dynamicMenus')
    this.dynamicMenus = []

    localStorage.removeItem('token')
    this.token = ''
      
    this.message = ''
    this.state = 'pending'    
  }
}

export default new RoutesStore()
```

登录页面

```tsx
function A8Login() {
  function onFinish(values: { username: string; password: string }) {
    RoutesStore.login(values)
  }

  const nav = useNavigate()
  useEffect(() => {
    if (RoutesStore.state === 'done') {
      nav('/')
    } else if (RoutesStore.state === 'error') {
      message.error(RoutesStore.message)
    }
  }, [RoutesStore.state])

  // ...
}

export default observer(A8Login)
```

* 用 useNavigate() 返回的函数做跳转时，不能在函数组件的渲染主逻辑（函数体顶层）中直接调用，否则会在渲染期间触发路由更新，引起警告；只能放在
  * 事件处理函数中
  * 副作用函数 useEffect 之中



#### 注销、欢迎词、登录检查

Store 中增加 get username 方法

```tsx
class RoutesStore {
  // ...

  // eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.-l-MjMPGJVOf3zoIJgoqpV3LWoqvCCgcaI1ga86ismU
  get username() {
    if(this.token.length === 0) {
      return ''
    }
    const json = atob(this.token.split('.')[1])
    return JSON.parse(json).sub
  }
    
  // ...
}
```

* JWT 的 token 由 header.payload.signature 三部分组成，前两部分都是 Base64 编码的 JSON，可以用 atob 解码；其中 [1] 就是 payload（内容）部分，sub 中存的是用户名

主页组件改成

```tsx
import { Button, Layout, Menu } from 'antd'
import { observer } from 'mobx-react-lite'
import { useEffect } from 'react'
import { Navigate, Outlet, useNavigate } from 'react-router-dom'
import RoutesStore from '../store/RoutesStore'

function A8Main() {
  const nav = useNavigate()

  function onClick() {
    RoutesStore.reset()
    nav('/login')
  }

  /* useEffect(()=>{
    if(RoutesStore.username === '') {
      nav('/login')
    }
  }, []) */

  if(RoutesStore.username === '') {
    return <Navigate to='/login'></Navigate>
  }

  return (
    <Layout>
      <Layout.Header>
        <span>欢迎您【{RoutesStore.username}】</span>
        <Button size='small' onClick={onClick}>注销</Button>
      </Layout.Header>
      <Layout>
        <Layout.Sider>
          <Menu items={RoutesStore.menus} theme='dark' mode='inline'></Menu>
        </Layout.Sider>
        <Layout.Content>
          <Outlet></Outlet>
        </Layout.Content>
      </Layout>
    </Layout>
  )
}

export default observer(A8Main)
```

* 这个例子中推荐用 Navigate 来完成跳转
* /student，/teacher 等路由不需要检查，因为登录成功后才有



## 附录

### 代码片段

ctrl+shift+p 输入关键词**代码**

![image-20221026102533928](imgs/image-20221026102533928.png)

定义 fun.code-snippets

```json
{
	"函数组件": {
		"scope": "javascript,typescript,typescriptreact",
		"prefix": "fun",
		"body": [
			"export default function ${1:函数名} () {",
      "  $0",      
      "  return <></>",
			"}"
		],
		"description": "快速生成react函数式组件"
	}
}
```

定义 ofun.code-snippets

```json
{
	"mobx函数组件": {
		"scope": "javascript,typescript,typescriptreact",
		"prefix": "ofun",
		"body": [
      "import { observer } from \"mobx-react-lite\"",
      "",
			"function ${1:函数名} () {",
      "  $0",      
      "  return <></>",
			"}",
      "export default observer($1)",
		],
		"description": "快速生成mobx react函数式组件"
	}
}
```

这样就能在 tsx 里用快捷键 `fun`、`ofun` 创建对应代码片段
