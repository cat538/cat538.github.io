## 页面布局设置
### 导航栏设置
通过设置`mkdocs.yaml`中的feature属性可以配置导航栏
```yaml
features:
    - navigation.sections
    # - navigation.tabs
```
其中 `navigation.sections` 会展开侧边栏的目录，我选择设置，认为可以更直观方便找文章
虽然 `navigation.expand` 会自动展开侧栏，达到类似的效果，但是感觉不太漂亮。

其中 `navigation.tabs` 会在header生成tab，我选择不设置

### 内容宽度设置

## 使用 Admonitions
**启用：**
```
markdown_extensions:
  - admonition
```
- **<u>尝试Note:</u>**
```
!!! note
    note admonition
```
!!! note
    note admonition
*设置标题：*
```
!!! note "这是一个标题"
    设置标题只需要在note后使用引号括住相应标题内容
```
!!! note "这是一个标题"
    设置标题只需要在note后使用引号括住相应标题内容
*标题置空：*
```
!!! note ""
    Empty title
```
!!! note ""
    Empty title
*默认类型：*
```
!!! 注意
    如果不指定fallback类型，默认为note类型，且标题被替换为无法识别的qualifier
```
!!! 注意
    如果不指定fallback类型，默认为note类型，且标题被替换为无法识别的qualifier

- **<u>尝试 inline block</u>**
!!! info inline end
    使用 `inline` + `end` 修饰Admonitions块可以使其变成一个内联块，并显示在右边，
或者只使用`inline` 修饰，这样会显示在最左边。

使用 `inline` + `end` 修饰Admonitions块可以使其变成一个内联块，并显示在右边，
或者只使用`inline` 修饰，这样会显示在最左边。

Admonitions, also known as call-outs, are an excellent choice for including side content without significantly interrupting the document flow. Material for MkDocs provides several different types of admonitions and allows for the inclusion and nesting of arbitrary content.

- 尝试其它类型
!!! tip
    tip
!!! abstract
    abstract
!!! example
    example
!!! question
    question
!!! quote
    quote cite
!!! warning
    warning
!!! error
    error

## 启用code block highlight
mkdocs中启用代码高亮有两种方式：during build time by Pygments or runtime with a JavaScript highlighter

默认不开启行号，如果要开启可以参考官方文档:[mkdocs document](https://mkdocs-material.zimoapps.com/reference/code-blocks/#highlight)。
经过测试显然是使用Pygments效果更好，启动方式：在mkdocs.yaml中添加(还有一些其它选项，下文是详细解释)
``` yaml
markdown_extensions:
  - pymdownx.highlight
  - pymdownx.superfences
```

- 其中pymdownx.superfences允许嵌套代码块的插入(默认竟然不允许···)
    
    可以使用 `hl_lines="2 3"`标识高亮line
    ```C++ hl_lines="2 3"
    #include<iostream>
    using namespace std;
    int main(){
        return 0;
    }
    ```

    ```rust
    let x: Vec<u8> = Vec::new()
    ```

- 还有个好玩的特性是 Grouping code blocks:

    使用这个功能需要在配置文件中加入:
    ```yaml
    - pymdownx.tabbed:
        alternate_style: true
    ```
    ```
    === "C"

        ``` c
        #include <stdio.h>
        int main(void) {
        printf("Hello world!\n");
        }
        ```

    === "C++"

        ``` c++
        #include <iostream>
        int main(void) {
        std::cout << "Hello world!" << std::endl;
        }
        ```
    ```

    === "C"

        ``` c
        #include <stdio.h>
        int main(void) {
        printf("Hello world!\n");
        }
        ```

    === "C++"

        ``` c++
        #include <iostream>
        int main(void) {
        std::cout << "Hello world!" << std::endl;
        }
        ```
    注意这不仅可以用于代码块group，其它类型的block也可以这样被组合在一起。只需要
    与上面的例子一样，将"C"，"C++"换成标题名字即可

- pymdownx.snippets允许从其它文件引入代码，感觉可能用的不多，没有开启
- pymdownx.inlinehilite比较有趣，添加了inline code highlight的feature
    开启该feature后，行内代码高亮通过使用shebang-like sequence,如`#!go var ch = 'a'`
    但是我认为没太大作用，因此不开启

## 表格

| Method      | Description                          |
| :----------- | :------------------------------------ |
| `GET`       | :material-check:     Fetch resource  |
| `PUT`       | :material-check-all: Update resource |
| `DELETE`    | :material-close:     Delete resource |
| 🐋         |   test emoji in table                |

## emoji和icon
据官网宣称，emoji和icon是MkDocs的一个亮点，支持的比较全(<s>这个年代还有支持不全的吗😓</s>)
在配置文件中添加:
```yaml
- pymdownx.emoji:
      emoji_index: !!python/name:materialx.emoji.twemoji
      emoji_generator: !!python/name:materialx.emoji.to_svg
```
接下来可以这样使用：
!!! example "emoji example"
    - `:material-github:` :material-github:
    - `:crab:` :crab:
    - `:whale:` :whale:

## MathJax支持
在配置文件中添加:
```yaml
markdown_extensions:
  - pymdownx.arithmatex:
      generic: true
```