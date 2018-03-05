# Jenkins

- [官网](http://www.jenkins.io)

## 简介
Jenkins是一个独立的开源自动化服务器，可用于自动化各种任务，如构建，测试和部署软件。 Jenkins可以通过本机系统软件包Docker安装，甚至可以通过安装Java Runtime Environment的任何机器独立运行。

## 导览
本导览将使用Jenkins发行版，它需要最少的Java 7，尽管推荐使用Java 8。 还建议使用512MB以上RAM的系统。
1. [下载](http://mirrors.jenkins.io/war-stable/latest/jenkins.war)
2. 在下载目录中打开终端并运行java -jar jenkins.war
3. 浏览到http：// localhost：8080，并按照说明完成安装。
4. 许多Pipeline示例需要在与Jenkins相同的计算机上安装Docker。

安装完成后，开始将Jenkins运行并创建Pipeline。Jenkins Piperline是一套插件，支持将连续输送Piperline实施并整合到Jenkins.Piperlin提供了一组可扩展的工具,用于将"simple-to-complex delivery pipelines"作为代码进行建模.

Jenkinsfile是一个包含Jenkins Piperline定义的文本文件，并被检入源代码控制。 这是“Pipeline-as-Code”的基础; 处理连续传送Piperline一部分应用程序，以便像其他代码一样进行版本和审阅。 创建Jenkinsfile提供了一些直接的好处：
- 自动为所有分支拉请求创建Piperline
- Piperline上的代码审查/迭代
- Piperline的审计跟踪
- Piperline的唯一代码来源，可以由项目的多个成员查看和编辑。

虽然在Web UI或Jenkinsfile中定义Piperline的语法是相同的，但通常认为最佳做法是在Jenkins文件中定义Piperline并检查源控制。
