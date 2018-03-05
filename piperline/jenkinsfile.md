# Jenkinsfile
创建一个Jenkinsfile,它被检入源代码控制.这样可以提供一些好处:
- pipeline上的代码审查/迭代
- pipeline的审计跟踪
- 管道的唯一真实来源,可以由项目中的多个成员查看和编辑(便于团队合作).

## 创建Jenkinsfile
如“入门”部分所述，Jenkinsfile是一个包含Jenkins管道定义的文本文件，并被检入源代码控制。 考虑以下pipeline，实施基本的三阶段连续输送pipeline。
```Groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Building..'
            }
        }
        stage('Test') {
            steps {
                echo 'Testing..'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying....'
            }
        }
    }
}
```
并非所有的管道都将具有相同的三个阶段，但是对于大多数项目来说，这是一个很好的起点。 以下部分将演示在Jenkins的测试安装中创建和执行简单的pipeline。
>假设已经有一个项目的源码管理库，并且已经在Jenkins中按照说明定义了一个pipeline

使用文本编辑器，理想的是支持Groovy语法高亮显示的文本编辑器，在项目的根目录中创建一个新的Jenkins文件。

上述声明性管道示例包含实现连续传送piperline的最小必要结构。 需要的代理指令指示Jenkins为piperline分配一个执行器和工作区。 没有agent指令，piperline声明无效，所以不能做任何工作！ 默认情况下，agent指令确保源已被检出，并可用于后续阶段的步骤

对于有效的声明性管道，Stage指令和steps指令也是必需的，因为它们会指示Jenkins执行什么以及应在哪个阶段执行。

对于脚本管道更高级的使用，上面的示例节点是分配管道的执行器和工作空间的关键第一步。 实质上，没有节点，管道不能做任何工作！ 从节点内部，业务的第一个顺序是checkout此项目的源代码。 由于Jenkins文件被直接从源代码控制中pull，所以Pipeline提供了一种快速简单的方法来访问源代码的正确版本
```Groovy
node {
    checkout scm
    /* .. snip .. */
}
```
chechout步骤将从源代码控制中检出代码; scm是一个特殊变量，指示检出步骤克隆触发此pipeline运行的特定版本。

## Build
对于大多数项目而言,piperline的"工作"的开始就是"build"阶段.通常,管道的这个阶段将是源码组装,编译或打包的地方.Jenkinsfile不能替代现有的构建工具,例如GNU/Make/Maven/Gradle等.而是将其视为粘合层来绑定项目开发生命周期的多个阶段(build,test,deploy,etc).

Jenkins有一些插件,用于调用几乎任何通用的构建工具,但是这个例子将简单的从shell步骤(sh)调用Make.sh步骤假定系统是基于Unix/Linux的.
```Groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'make'
                archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
            }
        }
    }
}
```
- sh步骤调用make命令，只有当命令返回值为0时才会继续。 任何非零退出代码将失败piperline。
- archiveArtifacts捕获与include模式（** / target / *。jar）匹配的文件，并将其保存到Jenkins主数据库以供以后检索。
> archiveArtifacts不能替代使用诸如Artifactory或Nexus之类的外部工件存储库，只能用于基本报告和文件归档。

## Test
运行自动化测试是任何成功的连续传送过程的重要组成部分。 因此，Jenkins有许多插件提供的测试记录，报告和可视化设备。 在基本层面上，当有测试失败时，让Jenkins在Web UI中记录报告和可视化的故障是有用的。 下面的示例使用由JUnit插件提供的junit步骤。

在下面的示例中，如果测试失败，则管道被标记为“不稳定”，如Web UI中的黄色球。 根据记录的测试报告，Jenkins还可以提供历史趋势分析和可视化。
```Groovy
pipeline {
    agent any

    stages {
        stage('Test') {
            steps {
                /* `make check` returns non-zero on test failures,
                * using `true` to allow the Pipeline to continue nonetheless
                */
                sh 'make check || true'
                junit '**/target/*.xml'
            }
        }
    }
}
```
- 使用内联条件（sh'make || true）确保sh步骤始终看到零退出代码，使junit步骤有机会捕获和处理测试报告。 下面的“处理故障”部分将详细介绍其他方法。
- junit捕获并关联与包含模式匹配的JUnit XML文件（** / target / *。xml）。

## Deploy
部署可能是步骤的各种变体,具体取决于项目或者组织的要求,可能是从构建的artifacts发送到artifacty服务器,将代码推送到生产系统的任何步骤.

在piperline示例的这个阶段，“构建”和“测试”阶段都已成功执行。 实际上，“部署”阶段只能在上一阶段成功完成，否则管道将早退。
```Groovy
pipeline {
    agent any

    stages {
        stage('Deploy') {
            when {
              expression {
                currentBuild.result == null || currentBuild.result == 'SUCCESS'
              }
            }
            steps {
                sh 'make publish'
            }
        }
    }
}
```
访问currentBuild.result变量允许管道确定是否有任何测试失败。 在这种情况下，该值将是“不稳定”。
> 假设一切都在Jenkins Pipeline示例中成功执行，每个成功的Pipeline运行都会将存档的关联构建工件，报告的测试结果和完整的控制台输出全部放在Jenkins中。
> Script Piperline可以包括条件测试（如上所示），循环，try / catch / finally块甚至函数。 下一节将详细介绍这种高级脚本管道语法。

## Piperline高级语法
### 字符串插值
Jenkins管道使用与Groovy相同的规则进行字符串插值。 Groovy的字符串插值支持可能会让很多新来的语言感到困惑。 虽然Groovy支持使用单引号或双引号声明一个字符串，例如：
```Groovy
def singlyQuoted = 'Hello'
def doublyQuoted = "World"
```
只有第二个字符串支持基于美元符号（$）的字符串插值，例如：
```Groovy
def username = 'Jenkins'
echo 'Hello Mr. ${username}'
echo "I said, Hello Mr. ${username}"
```
将会输出结果:
```Groovy
Hello Mr. ${username}
I said, Hello Mr. Jenkins
```
了解如何使用字符串插值对于使用一些管道更高级的功能至关重要。

## 环境变量
Jenkins管道通过全局变量env公开环境变量，该变量可从Jenkins文件中的任何位置获得。 假设Jenkins主机在localhost：8080上运行，那么在本地主机：8080 / pipeline-syntax / globals＃env中记录了可从Jenkins Pipeline中访问的环境变量的完整列表，其中包括：
### BUILD_ID
当前版本ID，与Jenkins版本1.597+中创建的builds相同，为BUILD_NUMBER
### JOB_NAME
此构建项目的名称，如“foo”或“foo / bar”。
### JENKINS_URL
完整的Jenkins网址，例如example.com:port/jenkins/（注意：只有在“系统配置”中设置了Jenkins网址时才可用）

参考或使用这些环境变量可以像访问Groovy中的Map任何key一样完成，例如：
```Groovy
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo "Running ${env.BUILD_ID} on ${env.JENKINS_URL}"
            }
        }
    }
}
```

## 设置环境变量
Declarative和Scripted Pipeline两种语法，在Jenkins piperline中设置环境变量是不同的。
Declarative Piperline支持environment指令，而Scripted Pipeline的用户必须使用withEnv步骤。
```Groovy
pipeline {
    agent any
    environment {
        CC = 'clang'
    }
    stages {
        stage('Example') {
            environment {
                DEBUG_FLAGS = '-g'
            }
            steps {
                sh 'printenv'
            }
        }
    }
}
```
- 顶层piperline块中使用的environment指令将适用于piperline中的所有步骤。
- 在stage中定义的environment指令将仅将给定的环境变量应用于stage中的步骤。

## 参数
声明式管道支持开箱即用的参数，允许管道在运行时通过parameters指令接受用户指定的参数。 使用脚本管道配置参数通过properties步骤完成，可以在代码段生成器中找到。

如果您使用“Build with Parameters”选项配置管道来接受参数，那么这些参数可以作为params变量的成员访问。

假设在Jenkins文件中配置了一个名为“Greeting”的String参数，它可以通过$ {params.Greeting}访问该参数：
```Groovy
pipeline {
    agent any
    parameters {
        string(name: 'Greeting', defaultValue: 'Hello', description: 'How should I greet the world?')
    }
    stages {
        stage('Example') {
            steps {
                echo "${params.Greeting} World!"
            }
        }
    }
}
```

声明式管道默认通过其post部分支持强大的故障处理，允许声明许多不同的“后置条件”，例如：always，unstable，success，failure和changed。 “Piperline语法”提供了有关如何使用各种条件的更多详细信息。
```Groovy
pipeline {
    agent any
    stages {
        stage('Test') {
            steps {
                sh 'make check'
            }
        }
    }
    post {
        always {
            junit '**/target/*.xml'
        }
        failure {
            mail to: team@example.com, subject: 'The Pipeline failed :('
        }
    }
}
```
> 然而脚本管道依赖于Groovy内置的try / catch / finally语义来处理管道执行过程中的故障。

>在上面的测试示例中，sh步骤被修改为从不返回非零退出代码（sh'make check || true'）。 这种方法虽然有效，但是意味着以下阶段需要检查currentBuild.result以了解是否有测试失败。

>处理这种情况的另一种方法是保留管道中的早期退出行为，同时仍然给予junit机会捕获测试报告，是使用一系列try / finally块


## 使用多代理
在所有以前的例子中，只使用了一个代理。 这意味着Jenkins将分配一个可用的执行节点，无论它是如何标记或配置的。 不仅可以覆盖此行为，而且Pipeline可以在同一个Jenkins文件中使用Jenkins环境中的多个代理，这对于更多的高级用例（如在多个平台上执行构建/测试）是有用的。

在下面的示例中，“build”阶段将在一个代理上执行，并且构建的结果将在“Test”阶段中分别标记为“linux”和“windows”的两个后续代理程序中重用。
```Groovy
pipeline {
    agent none
    stages {
        stage('Build') {
            agent any
            steps {
                checkout scm
                sh 'make'
                stash includes: '**/target/*.jar', name: 'app' 
            }
        }
        stage('Test on Linux') {
            agent { 
                label 'linux'
            }
            steps {
                unstash 'app' 
                sh 'make check'
            }
            post {
                always {
                    junit '**/target/*.xml'
                }
            }
        }
        stage('Test on Windows') {
            agent {
                label 'windows'
            }
            steps {
                unstash 'app'
                bat 'make check' 
            }
            post {
                always {
                    junit '**/target/*.xml'
                }
            }
        }
    }
}
```
- stash步骤允许捕获匹配包含模式（** / target / *。jar）的文件，以在同一个管道中重用。 一旦管道完成执行，垃圾文件将从Jenkins主站中删除。
- agent/node中的参数允许任何有效的Jenkins标签表达式。 有关详细信息，请参阅管道语法部分。
- unstash将从Jenkins主机中检索名为“stash”的管道的当前工作空间。
- bat脚本允许在基于Windows的平台上执行批处理脚本。

## 可选步骤参数
Piperline遵循Groovy语言约定，允许在方法参数中省略括号。
许Pinperline步骤还使用命名参数语法作为在Groovy中创建Map的简写，它使用语法[key1：value1，key2：value2]。 发表如下功能等同的语句：
```Groovy
git url: 'git://example.com/amazing-project.git', branch: 'master'
git([url: 'git://example.com/amazing-project.git', branch: 'master'])
```

为方便起见，当仅调用一个参数（或只有一个必需参数）时，可能会省略参数名称，例如：
```Groovy
sh 'echo hello' /* short form  */
sh([script: 'echo hello'])  /* long form */
```

## 高级脚本管道
Scripted Pipeline是基于Groovy的领域专用语言，大多数Groovy语法可以在脚本Piperline中使用而无需修改。

### 并行执行
上面的例子在线性系列中的两个不同平台上运行测试。 实际上，如果完成检查执行需要30分钟，“测试”阶段现在需要60分钟才能完成！

幸运的是，Pipeline具有内置的用于并行执行Scripted Pipeline的功能，并在适当命名的并行步骤中实现。
重构上述示例以使用parallel步骤：
```Groovy
stage('Build') {
    /* .. snip .. */
}

stage('Test') {
    parallel linux: {
        node('linux') {
            checkout scm
            try {
                unstash 'app'
                sh 'make check'
            }
            finally {
                junit '**/target/*.xml'
            }
        }
    },
    windows: {
        node('windows') {
            /* .. snip .. */
        }
    }
}
```
而不是在“linux”和“windows”标签的节点上执行测试，它们现在将在Jenkins环境中存在必需容量的情况下并行执行。
