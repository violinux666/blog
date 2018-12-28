> react-vio-form 是一个react的快速轻量表单库，能快速实现表单构建。提供自定义表单格式、表单校验、表单信息反馈、表单信息隔离等功能。可采用组件声明或者API的形式来实现表单的功能

***react-vio-form 基于React.createContext实现，要求开发者使用React16+的版本***

github:[地址](https://github.com/violinux666/react-vio-form)

## 安装

```
npm install --save react-vio-form
```

## 快速教程

首先我们先自定义自己的输入框组件

***InputGroup.js***
```
import React, { Component } from 'react';
class InputGroup extends Component {
  render() {
      let {
        onChange,//必须使用的属性，表单字段的值改变方法
        value,//必须使用的属性，表单字段的值
        message,////必须使用的属性,表单字段的报错信息
        title,//自定义属性
        type="text"//自定义属性
      }=this.props;
      return (
          <div>
              <label>{title}:</label>
              <input type={type} onChange={e=>onChange(e.target.value)}/>
              {message&&<span>{message}</span>}
          </div>
      );
  }
}
export default InputGroup;
```

接着我们配置我们的表格
- 最外层是Form组件，向它传递一个onSubmit的回调方法，在回调方法内我们输出表单的值。
- 里面是Field组件，它接收我们刚才写的InputGroup为component属性、fieldName为该字段的名称、regexp为该字段的校验正则表达式、message为当表单校验不通过的时候显示的报错信息，该信息通过props传递给InputGroup
- 一个简单列表demo就完成了，当在username或者password输入值或者点击Submit按钮就会触发表单的校验逻辑
***App.js***
```
import React, { Component } from 'react';
import InputGroup from './InputGroup';
let requiredExp=/\w{1,}/;
class App extends Component {
    handleSubmit=({model})=>{
        console.log('form data is :'+JSON.stringify(model));
    }
    render() {
        return (
            <Form onSubmit={this.handleSubmit}>
                <Field component={InputGroup} fieldName="username" title="Username" 
                regexp={requiredExp} message="Not be empty"></Field>
                <Field component={InputGroup} fieldName="address" title="Address"></Field>
                <Field component={InputGroup} fieldName="password" title="Password" 
                type="password" regexp={requiredExp} message="Not be empty"></Field>
                <button type="submit">Submit</button>
            </Form>
        );
    }
}
export default App;
```

## 回调函数

- ```<Form onSubmit={//}>```
    只有表单验证通过了才会触发
- ```<Field onChange={//}>```
    字段每次修改都会触发
```
class App extends Component {
    handleSubmit=({model})=>{
        //form submit callback
        console.log('form data is :'+JSON.stringify(model));
    }
    passwordChange=(value,{model,form})=>{
        //change callback
        //value：该字段的值
        //model：为整个表单的数据模型
        //form：表单的api中心
        console.log(`password:${value}`);
    }
    render() {
        return (
            <div>
                <Form onSubmit={this.handleSubmit} id="form">
                    <Field component={InputGroup} fieldName="username" title="Username"></Field>
                    <Field component={InputGroup} fieldName="password" title="Password" 
                    type="password" onChange={this.passwordChange}></Field>
                    <button type="submit">Submit</button>
                </Form>
            </div>
        );
    }
}
```

## API

表单实例form可以获取设置表单的信息，获取表单实例的方法有两种：
- formManager.get(id)
- 回调函数的参数

表单实例方法：
- setError(fieldName,message) 
- clearError(fieldName)
- getModel()
- submit()

```jsx
import React, { Component } from 'react'
import {Form,Field,formManager} from 'react-vio-form'
let requiredExp=/\w{1,}/;
class App extends Component {
    handleSubmit=({model})=>{
        //form submit callback
        console.log('form data is :'+JSON.stringify(model));
    }
    handleOutsideSubmit=()=>{
        // submit outside Form Component
        // param is Form id
        formManager.get('form').submit();
    }
    passwordChange=(value,{model,form})=>{
        if(model.password!==model.password2){
            //set Error Message
            form.setError('password2','password2 must be equaled to password');
        }else{
            //clear Error Message
            form.setError('password2','');
        }
    }
    render() {
        return (
            <div>
                <Form onSubmit={this.handleSubmit} id="form">
                    <Field component={InputGroup} fieldName="username" title="Username"></Field>
                    <Field component={InputGroup} fieldName="password" title="Password" 
                    type="password" regexp={requiredExp} 
                    message="Not be empty" onChange={this.passwordChange}></Field>
                    <Field component={InputGroup} fieldName="password2" title="Password2" 
                    type="password" onChange={this.passwordChange}></Field>
                </Form>
                <button onClick={this.handleOutsideSubmit}>outside submit</button>
            </div>
        );
    }
}
```

***持续更新中...***

## 反馈与建议
- 直接在github上提[issue](https://github.com/violinux666/react-vio-form/issues/new)吧 