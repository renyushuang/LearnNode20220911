## 掌握Gradle基础

### 环境配置

- 官网地址https://gradle.org/releases/
- 去官网下载gradle或者从本地用户文件夹下./gradle/wrapper/dists找到本地缓存的gradle开发工具包（注意带bin文件夹的这个gradle-x.x）
- 系统属性配置
  - 添加GRADLE_HONE: ...\wrapper\dists\gradle-6.5-all\gradle-6.5
  - 添加Path:%GRADLE_HONE%\bin

```
GRADLE_HOME=/Users/renyushuang/.gradle/wrapper/dists/gradle-3.3-all/ahd9f79pw091zdvevtf9phlqy/gradle-3.3
export GRADLE_HOME
export PATH=$PATH:$GRADLE_HOME/bin
```



### Hello Gradle!

- 在工程文件下，创建一个build.gradle文件

```
task hello{
	println "hello word !"
}
```

- 打开终端，移动到指定工程目录下，执行命令 gradle -q hello
- build.gradle是构建Project的核心文件，也是入口
  - 如果没有该文件，会出现not found in root project ‘’xxxx“提示异常
  - 必须要有一个可以运行的task，运行后自动生成.gradle文件夹下的内容。

### Gradle Wrapper

- Gradle Wrapper用来配置开发过程中用到的Gradle构建工具，避免因为Gradle不同意带来的不必要的问题
- 在工程目录下，使用cmd命令生成wrapper：gradle wrapper
- 标准的gradle工程目录
  - gradlew和gradlew.bat分别是linux和windows下的可执行脚本，具体业务逻辑是在/gradle/wrapper/gradle-wrapper.jar中实现，gradlew最终还是使用Java执行这个Jar包来执行相应的Gradle操作

### gradle-wrapper.properties

- 该配置 文件是gradle wrapper的相关配置
  - distributionBase：下载的Gradle压缩包解压后存储的主目录
  - distributionPath：相对于distributionBase的解压后的Gradle压缩包的路径
  - distributionUrl:Gradle发行版压缩包的下载地址
    - bin：二进制发布版。
    - all：bin基础上还包含了源码和文档
  - zipStoreBase：同distributionBase，只不过存放量是zip亚索巴的
  - zipStroePath ：同distributionPath，只不过存放的是zip压缩包的

### GradleUserHome

- 默认路径在～/.gradle/,不建议使用本地maven的m2替代，因为本地的.gradle目录下的模块分的很清晰，功能明确。
- 如果启动时，制定参数，使用别的目录替代GradleUserHome，后果是每次构建需要重新下载插件与依赖到新的项目
- 默认情况下，gradle运行时，除了和项目打交道，还有当前项目构建的全新GradleUserHome目录，所有Jar包都得重新下载

### Gradle命令行

- gradlew -？/-h/-help使用帮助
- gradlew tasks 查看所有可执行Tasks
- gradlew --refresh-dependencies assemble 强制刷新依赖
- gradlew cBC 等价与执行Task cleanBuildCache，这种通过缩写名快速执行任务。
- gradlew :app:dependencies 查找app工程依赖树



## Gradle构建机制

### Gradle标准工程

- .gradle
- .idea
- gradle
  - wrapper
    - Gradle-wrapper.jar
    - Gradle-wrapper.properties
- build.gradle
- gradlew
- gradlew.bat
- setting.gradle

### Java、Android工程目录

- .gradle
- .idea
- gradle
  - wrapper
    - Gradle-wrapper.jar
    - Gradle-wrapper.properties
- src
  - mian
    - java
    - resource
  - test
- build.gradle
- gradlew
- gradlew.bat
- setting.gradle



- .gradle
- .idea
- gradle
  - wrapper
    - Gradle-wrapper.jar
    - Gradle-wrapper.properties
- app
  - src
    - mian
      - java
      - resource
  - build.gradle
  - Proguadle-wrapper.properties
  - test
- build.gradle
- gradlew
- gradlew.bat
- setting.gradle

### Gradle DSL

- DSL(Domain Specific Language) 领域特定语言或领域专属语言。简单来说就是专门关注某一领域的语言，它在于专而不是全，组典型的比如HTML
- Gradle可以使用Groovy DSL，专门用来开发Gralde构建脚本。所以Gradle整体设计是以作为一种语言为导向的，并非成为一个严格死板的框架

### settings.gradle

- Gradle支持多工程构建，使用setting.gradle来配置添加自工程（模块）。
- settings文件在初始化阶段执行，创建Settings对象，在执行脚本时调用该对象的方法
- Setting.include(String...projectPaths)
  - 将给定的目录添加到项目构建中，“：app”表示文件相对路径，相当于“./app”文件夹
  - 多项目架构进行分层，把同层次的子工程放在同一文件夹下便于管理，使用“：xxx：yyy”表示

### build.gradle

- build.gradle是项目构建文件，每个工程都有一个build.gradle文件
- build.gradle在配置阶段执行，并创建相应的工程的project对象，执行的代码可以直接调用该对象提供的方法和属性。（project后面讲到Gradle核心模型时再进专门介绍）



### Deamon（守护进程）

- 项目启动时，会开启一个client，然后启动一个deamon，通过client向deamon收发请求，项目关闭，client关闭，deamon保持启动，由类型项目再次部署时，会直接通过新的client访问已经启动的deamon，所以速度很快，默认deamon不实用3小时后关闭；不同项目兼容性考虑，也可使用--no--deamon启动项目，就没有速度优势了
- 动手停止deamon:gradle wrapper --stop

### Gradle生命周期

- Initialization
  - Grad了支持单项目和多项目构建。在初始化阶段，gradle确定哪些项目将参与构建，并未每一个项目创建Porject实例，一般我们不会接触到它。（比如解析settings.gradle）
- Configation
  - 配置阶段，解析每个工程的build.gradle文件，创建要执行的任务子集和确定各种任务之间的关系，并对任务做一些初始化配置
- Execution
  - 运行阶段，Gradle根据配置阶段创建和配置要执行的任务子集，执行任务



### Gradle执行流程

初始化任务

Gradle.buildStarted—》执行settings.gradle—》Gradle.settingsEvaluated—》Gradle.projectLoaded（）

配置阶段

Gradle.beforeProject｜ Project.beforeEvaluate —》执行build.gradle—〉GradleafterProject｜ Project.afterEvaluate —》Gradle.projectsEvaluated | Gradle.taskGraph.whenReady

执行阶段

Gradle.taskGraph.beforeTask—》执行Task中的Actions—》Gradle.taskgraph.afterTask—》Gradle.buildFinish



### Config阶段

- 解析每个project中的build.gradle,解析过程中并不会执行各个build.gradle中的task
- 经过configration阶段，Project之间及内部Task之间的关系就确定了一个Porject包含很多个Task，每个Task之间有依赖关系。COnfiguration会建立一个有向图来描述Task之间的依赖关系，所有project配置完成后，会有一个回调project.afterEvaluate,表示所有的模块已经配置完毕了。



## Gradle 任务

### Task

- build.gradle中自定义任务
  - task <任务名>{...},在Gradle5.x以上已经删除<<操作符这种写法
  - {...}执行的是配置阶段的代码，执行阶段要处理的逻辑需要调用doFirst、doLast方法，在闭包中实现。doFirst{}表示任务执行开始时调用的方法，doLast{}表示任务执行结束调用的方法
  - task A(dependsOn:[B])表示任务A依赖于任务B，那么B执行在A之前
  - 自定义的任务默认分组到other中。

### 自定义任务类

- task定义的任务其实就是DefaultTask的一种具体实现类的对象



