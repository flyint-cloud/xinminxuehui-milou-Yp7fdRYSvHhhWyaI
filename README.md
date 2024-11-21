
Hvigor允许开发者实现自己的插件，开发者可以定义自己的构建逻辑，并与他人共享。Hvigor主要提供了两种方式来实现插件：基于hvigorfile脚本开发插件、基于typescript项目开发。下面以基于hvigorfile脚本开发插件进行介绍。


## 基于hvigorfile脚本开发


基于hvigorfile.ts脚本开发的方式，其优点是可实现快速开发，直接编辑工程或模块下hvigorfile.ts即可编写插件代码，不足之处是在多个项目中，无法方便的进行插件代码的复用和共享分发。


1. 导入模块依赖。



```
// 导入接口
import { HvigorPlugin, HvigorNode } from '@ohos/hvigor'

```

2. 编写插件代码。
在hvigorfile.ts中定义插件方法，实现HvigorPlugin接口。



```
// 实现自定义插件
function customPlugin(): HvigorPlugin {
    return {
        pluginId: 'customPlugin',
        apply(node: HvigorNode) {
            // 插件主体
            console.log('hello customPlugin!');
        }
    }
}

```

3. 在导出声明中使用插件。



```
export default {
    system: appTasks,
    plugins:[
        customPlugin()  // 应用自定义Plugin
    ]
}

```

## 使用hvigorfile插件动态生成navigation防混淆文件


我们在使用navigation的系统路由表时，每次添加新页面，都需要配置一下release环境防混淆。若将这些页面放在一个固定的目录下，则与我们的模块化设计相违背，若命名使用固定的前缀或后缀，总感觉有点多余，手动一个一个的添加，虽然符合我们的代码规范设计，但就是有点繁琐。有没有更方便的方式来处理这个混淆配置呢？


其实我们可以在写一个hvigorfilew插件来自动生成混淆配置文件。我们自定义一个HvigorPlugin任务，通过OhosHapContext对象读取module.json5文件中的routerMap字段，可以获取系统路由表的名称，再读取profile目录下的路由表。解析json文件内存，并将页面路径写到一个混淆文件中，这样每次编译时，自动生成防混淆文件，我们只需要引入这个文件就可以了。示例如下



```
import { hapTasks, OhosHapContext, OhosPluginId } from '@ohos/hvigor-ohos-plugin'
import { HvigorPlugin, HvigorNode, FileUtil } from '@ohos/hvigor'

function parseRouterMap(): HvigorPlugin {
  return {
    pluginId: 'parseRouterMap',
    apply(node: HvigorNode) {
      const hapCtx = node.getContext(OhosPluginId.OHOS_HAP_PLUGIN) as OhosHapContext
      const moduleJson = hapCtx.getModuleJsonOpt()
      const routerMapName = moduleJson['module']['routerMap'].split(':')[1]
      const dir = hapCtx.getModulePath()
      const srcFile = FileUtil.pathResolve(dir, 'src', 'main', 'resources', 'base', 'profile', `${routerMapName}.json`)
      const json = FileUtil.readJson5(srcFile)
      const routerRuleFile = FileUtil.pathResolve(dir, 'obfuscation-router.txt')
      FileUtil.ensureFileSync(routerRuleFile)
      const routerMapArray = json['routerMap']
      let rules = '-keep-file-name\n'
      for (const element of routerMapArray) {
        const pageSourceFile = element['pageSourceFile']
        const path = pageSourceFile.substring(0, pageSourceFile.lastIndexOf('.'))
        rules += `${path}\n`
      }
      FileUtil.writeFileSync(routerRuleFile, rules)
    }
  }
}

export default {
  system: hapTasks,
  plugins:[parseRouterMap()]
}

```

编译后会在entry目录下生成obfuscation\-router.txt防混淆文件，只要引入这个文件就可以了。


## 使用hvigorfile插件动态生成navigation页面枚举名称


我们在我们navigation的push跳转到新页面时，都得提前定义好系统路由表中的页面name，因为使用的name与系统路由表中定义的name不相同时，跳转页面则会白屏。有了前面的经验，其它我们也可以动态生成一个ets文件，将系统路由表中的页面名称自动生成一个枚举，这样就不用每次配置系统路由表，还是复制一下名称了。例如我们的系统路由表是这样的



```
{
  "routerMap": [
    {
      "name": "dialog",
      "pageSourceFile": "src/main/ets/pages/dialog/DialogPage.ets",
      "buildFunction": "dialogBuilder"
    },
    {
      "name": "web",
      "pageSourceFile": "src/main/ets/pages/web/WebPage.ets",
      "buildFunction": "webBuilder"
    },
    {
      "name": "login",
      "pageSourceFile": "src/main/ets/pages/login/LoginPage.ets",
      "buildFunction": "loginBuilder"
    }
  ]
}

```

我们现在实现一个hvigorfile插件，来解析系统路由表中的name字段，并生成对应的枚举值。示例如下



```
import { hapTasks, OhosHapContext, OhosPluginId } from '@ohos/hvigor-ohos-plugin'
import { HvigorPlugin, HvigorNode, FileUtil } from '@ohos/hvigor'

function parseRouterMap(): HvigorPlugin {
  return {
    pluginId: 'parseRouterMap',
    apply(node: HvigorNode) {
      const hapCtx = node.getContext(OhosPluginId.OHOS_HAP_PLUGIN) as OhosHapContext
      const moduleJson = hapCtx.getModuleJsonOpt()
      const routerMapName = moduleJson['module']['routerMap'].split(':')[1]
      const dir = hapCtx.getModulePath()
      const srcFile = FileUtil.pathResolve(dir, 'src', 'main', 'resources', 'base', 'profile', `${routerMapName}.json`)
      const json = FileUtil.readJson5(srcFile)
      const routerMapFile = FileUtil.pathResolve(dir, 'src', 'main', 'ets', 'Pages.ets')
      FileUtil.ensureFileSync(routerMapFile)
      const routerMapArray = json['routerMap']
      let ss = ''
      for (const element of routerMapArray) {
        const name = element['name']
        ss += `  ${name} = '${name}',\n`
      }
      ss = `export enum Pages {\n${ss}}`
      FileUtil.writeFileSync(routerMapFile, ss)
    }
  }
}

export default {
  system: hapTasks,
  plugins:[parseRouterMap()]
}

```

我们在ets目录下生成了一个Pages.ets文件，并将所有navigation页面生成对应的枚举值，页面跳转时，使用这些枚举值就不怕出错了。Pages.ets内容如下



```
export enum Pages {
  dialog = 'dialog',
  web = 'web',
  login = 'login',
}

```

 本博客参考[FlowerCloud机场](https://hushicha.org)。转载请注明出处！
