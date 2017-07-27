# Piperline语法
## 声明式Piperline
声明式Piperline是Jenkins Pipeline [1]的一个相对较新的补充，它在管道子系统之上提出了一种更为简化和有意义的语法。
所有有效的声明性管道必须包含在Piperline块中，例如：
```Groovy
pipeline {
    /* insert Declarative Pipeline here */
}
```
声明式Piperline中有效的基本语句和表达式遵循与Groovy语法相同的规则，但有以下例外：
- Piperline的顶层必须是一个块，具体来说是：pipeline {}
- 没有分号作为语句分隔符。 每个声明必须在自己的一行
- 块只能包含Sections, Directives, Steps或赋值语句。
- 属性引用语句被视为无参数方法调用。 所以例如，输入被视为input（）

## Sections
声明式Piperline中的Section通常包含一个或多个Directives或Steps。

### agent
代理部分指定整个流水线或特定阶段将在Jenkins环境中执行的位置，具体取决于代理部分的放置位置。 该段必须在流水线块内部的顶层定义，但级别使用是可选的。

必要性: Yes
参数: 如下
许可范围: 在pipeline顶级块和每个阶段块
#### 参数
为了支持pipeline使用者可能拥有的各种用例，agent部分支持几种不同类型的参数。 这些参数可以应用在pipeline块的顶层，也可以应用在每个阶段的指令中。
##### any
可以在任何适合的代理上执行Pipeline或者stage. 例子: agent any
##### none
当在pipeline块的顶层应用时,将不会为整个pipeline分配全局agent,并且每个stage部分将需要包含其自己的代理部分. 例子:agent none
##### label
在label提供的Jenkins环境中的可用的agent上执行pipeline或者stage. 例子: agent label{lable 'my-defined-label'}
##### node
agent {node {lable "lableName"}}的行为与agent {lable "lableName"}相同,但node允许其他选项(例如customWorkspace)
##### docker
使用指定的docker容器执行pipeline或stage,该容器将在有docker环境的pipeline的节点上或可选定义的lable参数匹配的节点上动态配置. docker还可用选择接受一个args参数,该参数可能包含直接传递给docker运行调用的参数.例子:agent{docker 'maven:3-alpine'}或者
```Groovy
agent {
    docker {
        image 'maven:3-alpine'
        label 'my-defined-label'
        args  '-v /tmp:/tmp'
    }
}
```
##### dockerfile
使用从source repository中包含的Dockerfile构建的容器执行pipeline或stage.通常这是source repository根目录中的Dockerfile:agent{dockerfile true}.如果在其他目录中构建Dockerfile,请使用dir选项:agent{dockerfile {dir 'someSubDir'}}.你可用使用additionalBuildArgs选项将其他参数传递给docker build...命令,例子:agent{dockerfile {additionalBuildArgs '--build-arg foo=bar'}}

#### 常用选项
这些是可以应用两个及以上agent实现的几个选项。 除非明确说明，否则不需要(默认不需要)。
##### label
一个字符串。 运行pipeline或单独stage的标签。

##### customWorkspace
一个字符串. 在此自定义工作空间中运行该agent以运行该pipeline或单个stage,而不是默认值.它可用是相对路径,在这种情况下,自定义工作空间位于该节点上的工作空间根目录下,也可用是绝对路径.例如:
```Groovy
agent {
    node {
        label 'my-defined-label'
        customWorkspace '/some/other/path'
    }
}
```
该选项只对node,docker,dockerfile有效

##### reuseNode(重要节点)
一个布尔值,默认为false. 如果设置为true,则在同一工作空间中,而不是完全新的节点上运行pipeline顶层指定的节点上的容器

此选项对docker和dockerfile有效,并且仅在单个阶段的代理上使用才有效果.

#### 例子
```Groovy
pipeline {
    agent { docker 'maven:3-alpine' }
    stages {
        stage('Example Build') {
            steps {
                sh 'mvn -B clean verify'
            }
        }
    }
}
```
在新创建的给定名称和标签的容器（maven：3-alpine）中执行此流水线中定义的所有步骤。

####Stage级别的agent
```Groovy
pipeline {
    agent none 1
    stages {
        stage('Example Build') {
            agent { docker 'maven:3-alpine' } 2
            steps {
                echo 'Hello, Maven'
                sh 'mvn --version'
            }
        }
        stage('Example Test') {
            agent { docker 'openjdk:8-jre' } 3
            steps {
                echo 'Hello, JDK'
                sh 'java -version'
            }
        }
    }
}
````
- 1:在管道顶层定义agent none，确保不会不必要地分配执行者。 使用agent none也强制每个阶段部分包含自己的代理部分。
- 2:使用此镜像在新创建的容器中执行此阶段中的步骤。
- 3:使用前一个阶段的不同镜像创建的新的容器中执行此阶段中的步骤。

### Post
Post部分定义将在piperline运行或stage结束时运行的操作.后期部分支持许多后置条件块:always,changed,failure,success和unstable.这些块允许在pipeline运行或stage结束时执行steps,具体取决于piperline的状态.


必要性:否
参数:无
允许: pipeline顶层块,每个stage

#### 条件
##### always
无论管道运行的完成状态如何都运行。
##### changed
只有当前piperline的运行状态和先前完成的pipeline状态不同时,才运行.
##### failure
仅当当前pipeline处于"failed"状态时才运行,通常在Web UI中用红色表示
##### success
仅当当前的piperline处于"success"状态时才运行,通常在Web UI中用蓝色或绿色表示
##### unstable
仅当当前的piperline处于"unstable"状态时才运行,通常由测试失败，代码违例等引起,通常在Web UI中用黄色表示

#### 例子
```Groovy
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
    post { 1
        always { 2
            echo 'I will always say Hello again!'
        }
    }
}
```
- 1: 通常情况下，post部分应放在pipelin末端。
- 2: 状态块包含有跟steps部分相同的功能的步骤。

