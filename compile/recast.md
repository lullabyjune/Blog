# 一些编译相关的tips

## 简单谈一下编译
编译可以简单的分成几个步骤：
> 1. 词法分析
> 2. 语法分析
> 3. 语义分析 & 中间产物ast
> 4. 优化
> 5. 产出目标代码

以`const a = 3`为例， 首先`1)`的输入是`源代码字符串`， 产出是`const, a, =, 3`这样一个一个的`词(token)`， 再此基础上进行`2)`, 输入就是上一阶段产生的`token`， 会将`token`按照语义进行组合，区分， 比如这个例子中， 所有的`token`就可以分在一起作为一个赋值语句。之后进行语义分析 & 产出ast， 这里产出的`ast`实际上而言是语言无关的， 即标准化的， 也就是说只要拿到一个ast， 理论上来说是可以转化成任意语言的相对应的代码的。 之后一般还会对`ast`进行一些优化， 最后产出目标代码。
> [参考链接](https://blog.csdn.net/cflys/article/details/71274116)

## recast
[项目地址](https://github.com/benjamn/recast)， 这个库主要是封装了`js`的编译过程， 作为使用方而言， 可以直接调用`api`去操作`ast`， 可以简单的做到`code -> ast -> code`的转换（其实`babel`也可以？

主要api就俩， `recast.parse(code, options), recast.print(ast, options)`。前者通过读取源代码产出`ast`， 后者通过`ast`输出代码

### recast.parse(code, options)
`code: string`, 这个应该是符合预期的， 主要说的一点是， 这个库是支持`ts`的， 但是需要在`options.parser`中指定， 针对`ts`的分析， 具体使用如下：
```typescript
const recast = require('recast');
const tsParser = require('recast/parsers/typescript');

recast.parse(fs.writeFileSync(path), {
  parser: tsParser  //  其实这里还可以指定其他的parser， 比如babel的Babylon之类的
})
```
产出就是`ast`了， 结构大概如下
```json
{
  "program": {
    "type": "Program",
    "body": [ //  主要的ast节点部分
      {
        "type": "ExportNamedDeclaration", //  语句类型， 以具名导出为例 源码: export { name } from '../a.ts'
        "declaration": null,  //  声明
        "specifiers": [{}], //  导出的key， 即关键字
        "source": {}, //  导出的源文件， 即从哪导出，
        "loc": {} //  location， 即代码在source文件中的位置， 包括start， end， lines之类的信息
      }
    ],
    ...
  }
}
```

[更多type类型点击](https://github.com/yacan8/blog/blob/master/posts/JavaScript%E6%8A%BD%E8%B1%A1%E8%AF%AD%E6%B3%95%E6%A0%91AST.md)

在进行操作的时候， 主要也就是操作这个`body`的数组， 通过`react.types.builder`中的各个`builder`， 可以生成一些`ast`节点， 比如我生成一个`identifier`节点`push`到`body`中， 那么最后在`print`的时候， 就会多出来一个东西（实际上单纯只`push`一个`identifier`没啥意义， 会不会产出代码没有实践过=。=， `identifier`一般是作为参数提供给其他的`builder`.

举个例子:
```javascript
body.unshift(
  importDeclaration(
    values.map(value => importSpecifier(identifier(value))),
    literal(filePath),
  ),
);
```
上面的代码是在源文件的顶部插入`import`代码，产出类似于这样`import { a, b, c } from 'path'`， 其实也比较好理解， 因为要生成一个具名`import`语句的话， 需要的两个必要参数就是`变量名 & 导入路径`， `导入路径`很明显会是一个`literal`的字面量， 而`变量名`首先是一个`importSpecifier`， 其次才是一个`identifier`

这个是直接在原来的body上进行增删的操作， 如果说是要在原来的body中进行改， 并且不想自己去遍历`ast`的话， 可以这样
```javascript
recast.visit(ast, {
  visitTSTypeAliasDeclaration(nodePath) {
    if (
      nodePath.value &&
      nodePath.value.id &&
      nodePath.value.id.name === 'TTest'
    ) {
      nodePath.value.typeAnnotation = tsIntersectionType(
        Array.from(typeNames).map(typeName =>
          tsTypeReference(identifier(typeName)),
        ),
      );
    }

    this.traverse(nodePath);  //  这个是语法要求， 最后必须要返回this.traverse() 或者 return false
    //  return false的话会停止ast的遍历， 可以用在确定只会操作一次的场景下， this.traverse的话则是会继续后面的遍历
  }
})
```
上述的代码是库本身提供的api， 用来遍历`ast`， 具体可`visit`的节点类型， 可以参照上面的链接中的`type`类型， 上面有的类型都可以通过加一个`visit`来进行遍历
比如上面的代码， 就通过遍历`ast`中的`TypeAliasDeclaration`， 即`类型别名声明`节点， 并且在节点中进行判断， 针对`type TTest = '....'`这样的节点进行了改造， 将其改造成了`type TTest = A & B & C`这样的`交叉类型的类型别名声明语句`。

所以可以简单的通过上述两个`api`，完成一个小demo， 即从所有的`index.ts`导出文件中， 抽取所有的`export { name } from 'filePath'`这样的语句中的`name & path`， 并且在某个文件中重写类型文件， 或者批量导入的语句。 这样的场景是肯定存在的， 尤其是针对一些工具类库而言， 但是如果频繁使用的话其实会很蛋疼， 所以可以通过`recast`写一个脚本， 比如在`build`或者`dev`之前`run`一下这个脚本就可以取代人工操作了=。=


### recast.print(ast, options)
就是纯粹的从ast中产出代码， 与此类似的还有`recast.prettyPrint(ast, options)`， 用来产出比较好看的代码- -， 至少可以确保缩进， 比如`recast.prettyPrint(ast, {tabWidth: 2})`