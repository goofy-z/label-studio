# atta-ai-label-studio
基于开源项目[label-studio](https://github.com/heartexlabs/label-studio/tree/1.7.3)的1.7.3版本做的二次开发，并原版本上尽量做最小改动来实现功能。

## 项目架构
### 前端
- node版本：14
- 框架：主要是react，其中组织模块有部分用到vue
- ui组件：该组织自己封装的前端组件
### 后端：django
- python版本：`>=3.7 <=3.9`
- 数据库：改造后使用mysql
- 缓存：redis，这个主要是给django-rq做任务异步执行

## 改造功能
### 1. 配置改造
目前算法平台有部分隐私配置是在apollo上，例如：oss相关，数据库等。在`core/settings`下增加`apollo.py`来处理与apollo服务交互，同时新引入的配置统一加入到同目录下`label_studio.py`中
```
注：接入apollo后启动项目需要如下环境变量

"APOLLO_CONFIG_ENV": "PRO",
"APOLLO_CONFIG_URL": "http://10.12.0.243:40003",
"APOLLO_CONFIG_CLUSTER": "dev20",
"APOLLO_AUTH_TOKEN": "helloapollo"
```

### 2. 云存储支持oss
在`io_storages`模块下增加oss相关API，同时关闭现在支持的存储，只留下oss`io_storages/functions.py`，同时oss的`bucket`、 `ak`、`endpoint`, 都通过apollo配置获取

### 3. 权限改造

#### 角色分类
角色共分为三种：平台管理员、项目管理员、项目标注员，对应功能权限如下

| **功能**           | **平台管理员** | **项目管理员** | 项目标注员      |
| ------------------ | -------------- | -------------- | --------------- |
| 个人账号管理       | ✅              | ✅              | ✅               |
| 平台成员管理       | ✅              | ❌              | ❌               |
| 项目查看           | ✅              | ✅              | ✅（已分配项目） |
| 项目删除           | ✅              | ❌              | ❌               |
| 项目创建           | ✅              | ✅              | ❌               |
| 项目成员添加       | ✅              | ✅              | ❌               |
| 项目设置           | ✅              | ✅              | ❌               |
| 数据导入管理       | ✅              | ✅              | ❌               |
| 数据导出管理       | ✅              | ✅              | ❌               |
| 标注任务分配、删除 | ✅              | ✅              | ❌               |
| 任务查看           | ✅              | ✅              | ✅               |
| 任务标注           | ✅              | ✅              | ✅               |
| 任务标注结果管理   | ✅              | ✅              | ✅               |

#### 
#### 接口权限改造
1. 用户权限绑定
权限控制使用的django的auth扩展，[具体可以参考官方文档](https://docs.djangoproject.com/zh-hans/2.1/_modules/django/contrib/auth/)，简而言之就是基于角色的账号权限控制，`User`先绑定`Group`,`Group`再绑定每一个业务权限Code，对应的表如下:

| **表名**           | **描述** |
| ------------------ | -------------- | 
| auth_group       | 权限组，理解为角色            | 
| auth_group_permissions       | 权限组关联权限code            | 
| auth_permissions       | 权限表            | 
| user_group       | 用户表，当前项目user表被重命名为htx_user            |

同时平台初始化时需要初始化权限表和权限组以及前两者的关联表，这部分sql在`deploy/init/init_dml.sql`

2. 统一权限控制钩子
基于django的auth扩展，在对应API中指定路由使用的权限，通过统一的权限校验类`HasObjectPermission`, 这是通过drf里的配置`DEFAULT_PERMISSION_CLASSES`设置的，代码可以参考`core/api_permissions.py`。

#### 前端权限改造
1. 基于现在的前端渲染模式，用户的信息都是通过django模板渲染的，我们在`template/base.py`里将用户权限也一并返回, 在APP_SETTINGS里添加权限，调用的是用户的`get_all_permissions`方法
```
window.APP_SETTINGS = Object.assign({
  user: {
      ...
      permissions: "{{user.get_all_permissions}}",
      ...
    },
})
```
在react项目中可以使用如下方式来校验权限, 例如下面检查的是`projects.change`权限
```
import { useConfig } from '../providers/ConfigProvider';
  ...
  const config = useConfig()
  config.user.permissions.search("projects.change") > 0
  ...
```

2. 平台的标注数据管理组件的代码并不在当前项目里，而是该组织下的[另一个项目dm2](https://github.com/heartexlabs/dm2)，幸运的是这个组件是支持自定义按钮，详细可查看[调研文档](https://alidocs.dingtalk.com/i/nodes/Gl6Pm2Db8D3xp52rC6DgGbw1JxLq0Ee4?utm_scene=person_space)

