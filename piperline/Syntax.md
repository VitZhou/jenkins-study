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
范围: 在pipeline顶级块和每个阶段块
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
范围: pipeline顶层块,每个stage

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

### stages
包含一个或多个stage指令的序列，stages部分是由pipeline描述的大部分"工作"的阶段。 建议对于持续交付过程的每个阶段建立一个stage，如构建，测试和部署，stages至少包含一个stage指令。

必要性: YES
参数: 无
范围: 在pipeline块中只存在一次

#### 例子:
```Groovy
pipeline {
    agent any
    stages { 1
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```
1: stages部分通常将遵循directives，如agent，options等。

### steps
steps部分定义了在给定stage指令中执行的一个或多个步骤。
#### 例子
```Groovy
pipeline {
    agent any
    stages {
        stage('Example') {
            steps { 1
                echo 'Hello World'
            }
        }
    }
}
```
1: steps部分必须包含一个或多个步骤。

## Directives(指令)
### environment
environment指令指定一系列键值对，这些对值将被定义为所有步骤的环境变量或阶段特定的步骤，这取决于环境指令位于管道中的位置。
该指令支持一个特殊的帮助方法credentials（），可用于通过Jenkins环境中的标识符访问预定义的凭据。 对于类型为“Secret Text”的credentials，credentials（）方法将确保指定的环境变量包含Secret Text内容。 对于“标准用户名和密码”类型的credentials，指定的环境变量将被设置为username：password，另外两个环境变量将被自动定义：MYVARNAME_USR和MYVARNAME_PSW。

必要性:No
参数: none
范围: pipeline块内,或在stage指令内

#### 例子:
```Groovy
pipeline {
    agent any
    environment { 1
        CC = 'clang'
    }
    stages {
        stage('Example') {
            environment { 2
                AN_ACCESS_KEY = credentials('my-prefined-secret-text') 3
            }
            steps {
                sh 'printenv'
            }
        }
    }
}
```
- 1: 顶层pipeline块中使用的environment指令将适用于pipeline中的所有step。
- 2: 在stage中定义的环境指令将仅将给定的环境变量应用于stage中的步骤
- 3: 环境块有一个帮助方法credentials（）定义，可以用于在Jenkins环境中通过其标识符访问预定义的credentials。

### Options
options指令允许从管道本身中配置管道专用选项。 Pipeline提供了许多这样的选项，例如buildDiscarder，但是它们也可以通过插件来提供，例如时间戳timestamps。
范围: 在pipeline块内只允许出现一次.

#### 可用options
##### buildDiscarder
持久化artifacts和控制台输出,用于最近管道运行的具体数量。 例如：options {buildDiscarder（logRotator（numToKeepStr：'1'））}
##### disableConcurrentBuilds
不允许并行执行pipeline。 可用于防止同时访问共享资源等。例如：options {disableConcurrentBuilds（）}
##### skipDefaultCheckout
在代理指令中，默认跳过从源代码check out代码。 例如：options {skipDefaultCheckout（）}
##### skipStagesAfterUnstable
一旦stages状态进入了“UNSTABLE”状态，就跳过阶段。 例如：options {skipStagesAfterUnstable（）}
##### timeout
设置piperline运行的超时时间，超时之后Jenkins应该中止pipeline。 例如：options {timeout（time：1，unit：'HOURS'）}
##### retry
失败后，重试整个流水线指定的次数。 例如：options {retry（3）}
##### timestamps
预处理由pipeline生成的所有控制台输出运行时间与的时间。 例如：options {timestamps（）}
#### 例子
```Groovy
pipeline {
    agent any
    options {
        timeout(time: 1, unit: 'HOURS') 1
    }
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```
>1: 指定一个小时的全局执行超时时间，超时之后Jenkins将中止pipeline运行。

### parameters
parameters指令提供用户在触发管道时提供的参数列表。 这些用户指定参数的值通过params对象可用于管道steps，具体用法见示例。
范围: 在pipeline块内,只允许出现一次

#### 可用的parameters
##### string
一个字符串类型的参数，例如：parameters {string（name：'DEPLOY_ENV'，defaultValue：'staging'，description：''）}
##### booleanParam
一个布尔参数，例如：parameters {booleanParam（name：'DEBUG_BUILD'，defaultValue：true，description：''）}
#### 例子
```Groovy
pipeline {
    agent any
    parameters {
        string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
    }
    stages {
        stage('Example') {
            steps {
                echo "Hello ${params.PERSON}"
            }
        }
    }
}
```
### triggers
触发器指令定义了管道应重新触发的自动化方式。 对于与代码源（如GitHub或BitBucket）集成的管道，可能不需要触发器，因为基于webhook的集成可能已经存在。 目前只有两个可用的触发器是cron和pollSCM。
范围: pipeline内只允许出现一次
#### cron
接受一个cron风格的字符串来定义管道应重新触发的常规间隔，例如：triggers {cron（'H 4 / * 0 0 1-5'）}
#### PollSCM
接受一个cron风格的字符串来定义Jenkins应该检查新的代码源是否更新的常规间隔。 如果存在更新，则pipeline将被重新触发。 例如：triggers {pollSCM（'H 4 / * 0 0 1-5'）}
> pollSCM触发器仅在Jenkins 2.22或更高版本中可用。

#### 例子
```Groovy
pipeline {
    agent any
    triggers {
        cron('H 4/* 0 0 1-5')
    }
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```

### stage
stage指令进入steps部分，并应包含step部分，可选代理部分或其他特定于stage的指令。 实际上，管道完成的所有实际工作都将包含在一个或多个stage指令中。
必要性: 最少一个
可选参数: 一个强制参数，一个用于stage名称的字符串。
范围: stages部分内

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
}
```

### tools
定义自动安装和放置PATH的工具的部分。 如果指定了agent为none，则会被忽略。
范围: pipeline块或者stage块
#### 支持的工具
- maven
- jdk
- gradle

#### 例子
```Groovy
pipeline {
    agent any
    tools {
        maven 'apache-maven-3.0.1' 1
    }
    stages {
        stage('Example') {
            steps {
                sh 'mvn --version'
            }
        }
    }
}
```
> 1: 工具名称必须在Jenkins的Manage Jenkins→ Global Tool Configuration。

### when
when指令允许pipeline根据给定的条件确定是否执行该stage。 when指令必须至少包含一个条件。 如果when指令包含多个条件，则所有子条件必须返回true,stage才会执行。 这与子条件嵌套在allOf条件中相同（请参见下面的示例）。
可以使用嵌套条件构建更复杂的条件结构：not，allOf或anyOf。 嵌套条件可以嵌套到任意深度。
范围: 在stage指令内
#### 内置条件
##### branch
当正在构建的分支与给出的分支模式匹配时执行stage，例如：{branch'master'}。 请注意，这仅适用于多分支pipeline。
##### environment
当指定的环境变量设置为给定值时执行stage，例如：当{environment name：'DEPLOY_TO'，value：'production'}
##### expression
当指定的Groovy表达式求值为true时，执行阶段，例如：{expression {return params.DEBUG_BUILD}}
##### not
当嵌套条件为false时执行阶段。 必须包含一个条件。 例如：当{not {branch'master'}}
##### allOf
当所有嵌套条件都为真时，执行stage。 必须至少包含一个条件。 例如：{allOf {branch'master'; 环境名称：'DEPLOY_TO'，value：'production'}}
##### anyOf
当至少一个嵌套条件为真时执行stage。 必须至少包含一个条件。 例如：{anyOf {branch'master'; 分支'分期'}}

#### 例子
```Groovy
pipeline {
    agent any
    stages {
        stage('Example Build') {
            steps {
                echo 'Hello World'
            }
        }
        stage('Example Deploy') {
            when {
                branch 'production'
            }
            steps {
                echo 'Deploying'
            }
        }
    }
}
```
```Groovy
pipeline {
    agent any
    stages {
        stage('Example Build') {
            steps {
                echo 'Hello World'
            }
        }
        stage('Example Deploy') {
            when {
                branch 'production'
                environment name: 'DEPLOY_TO', value: 'production'
            }
            steps {
                echo 'Deploying'
            }
        }
    }
}
```
```Groovy
pipeline {
    agent any
    stages {
        stage('Example Build') {
            steps {
                echo 'Hello World'
            }
        }
        stage('Example Deploy') {
            when {
                branch 'production'
                anyOf {
                    environment name: 'DEPLOY_TO', value: 'production'
                    environment name: 'DEPLOY_TO', value: 'staging'
                }
            }
            steps {
                echo 'Deploying'
            }
        }
    }
}
```

### steps
声明性管道可以使用“pipeline step”引用中记录的所有可用步骤，其中包含一个完整的[步骤列表](https://jenkins.io/doc/pipeline/steps)，并附加以下列出的步骤，仅在声明性流水线中支持。

#### script
脚本步骤需要一个脚本管道块，并在声明性管道中执行。 对于大多数用例，声明式管道中脚本步骤不是必需的，但它可以提供一个有用的“转义填充”。 比较大或复杂性比较高的脚本块应该转移到共享库中。
```Groovy
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'

                script {
                    def browsers = ['chrome', 'firefox']
                    for (int i = 0; i < browsers.size(); ++i) {
                        echo "Testing the ${browsers[i]} browser"
                    }
                }
            }
        }
    }
}
```