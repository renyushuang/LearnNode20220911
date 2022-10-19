## Gradle核心模型

### task

- task是Gradle中最小的任务单元，任务之间可以进行复杂的操作（如动态创建任务，多任务间依赖调用等等）。Gradle的执行其实就是由各种任务组合执行，来对项目进行构建的
- 使用gradlew help命令，任何gradle项目都有一个该task，可以执行此命令观察task执行的流程是否如预期
- 可以使用工具查看，还可以通过 gradlew tasks命令查看可运行任务
  - 使用gradlew tasks --all 命令查看所有任务
  - 使用gradlew A B 命令表示执行任务A和B，支持驼峰简写



### 自定义任务

- build.gradle中自定义任务
  - task <任务名>{...},在Gradle5.x以上已经删除<<操作符这种写法
  - {...}执行的是配置阶段的代码，执行阶段要处理的逻辑需要调用doFirst、doLast方法，在闭包中实现。doFirst{}表示任务执行开始时调用的方法，doLast{}表示任务执行结束调用的方法
  - task A(dependsOn:[B])表示任务A依赖于任务B，那么B执行在A之前
  - 自定义的任务默认分组到other中。

### Default Task

- Task 定义在任务其实就是defaultTask的一种具体实现类的对像
- 可以使用自定义类继承DeaflutTask
  - 在方法使用上@TaskAction注解，表示任务运行时调用的方法
  - 使用@Input表示对任务的输入参数。
  - 使用@OutputFile表示任务的输出文件
  - 使用inputs、outputs直接设置任务输入/输出项
  - 一个任务的输出项可以作为另一个任务的输入项（隐式依赖关系）

### Gradle执行流程

初始化任务

Gradle.buildStarted—》执行settings.gradle—》Gradle.settingsEvaluated—》Gradle.projectLoaded（）

配置阶段

Gradle.beforeProject｜ Project.beforeEvaluate —》执行build.gradle—〉GradleafterProject｜ Project.afterEvaluate —》Gradle.projectsEvaluated | Gradle.taskGraph.whenReady

执行阶段

Gradle.taskGraph.beforeTask—》执行Task中的Actions—》Gradle.taskgraph.afterTask—》Gradle.buildFinish

### Gradle钩子函数

- gradle在生命周期三个阶段都设置了相应的钩子函数调用
- 使用钩子函数，处理自定义构建：
  - 初始化阶段gradle.settingsEvaluate和gradle.projects;oaded（在settings.gradle中生效）
  - 配置阶段：project.beforeEvaluate和project.afterEvaluate;
    Gradle.beforeProject、gradle.afterProject及Gradle.taskGraph.whenReady
  - 执行阶段Gradle.taskGraph.beforeTask和Gradle.taskgraph.afterTask

### gradle构建监听

- gradle可以设置监听，对各阶段都有相应的回调处理
  - gradle.addProjectEvaluationListener
  - Gradle.addBuildListener
  - Grade.addlistener:TaskEveexcutionGraphListener(任务执行图监听)、TaskExecutionListener（任务执行监听）、TaskExecutionListener、TaskActionListener、StandarOutputListener

### Deamon（守护进程）

- 项目启动时，会开启一个client，然后启动一个deamon，通过client向deamon收发请求，项目关闭，client关闭，deamon保持启动，由类型项目再次部署时，会直接通过新的client访问已经启动的deamon，所以速度很快，默认deamon不实用3小时后关闭；不同项目兼容性考虑，也可使用--no--deamon启动项目，就没有速度优势了
- 动手停止deamon:gradle wrapper --stop



### Porject

- build.gradle在配置阶段会生成project实例，在build.gradle中直接调用方法或属性，实则是调用当前工程的project对象的方法或属性
- 使用Projectr提供的API，再多项目设置游刃有余：
  - project(":app"){}制定的project（这里是app配置）
  - allproject{}在所有priject中配置
  - subproject{}所有的子project配置
  - buildscript{} 此项目配置构建脚本类路径

### 属性扩展

- 使用ext对任意对象属性进行扩展

  - 对project进行使用ext进行属性扩展，对所有子project可见
  - 一般在rootproject中进行ext属性扩展，为子工程提供复用属性，通过rootproject直接访问
  - 任意对象都可以使用ext来添加属性：使用闭包，在闭包中定义扩展属性。直接使用=赋值，添加扩展属性
  - 由谁进行ext的调用，就属于谁的扩展属性
  - 在build.gradle中，默认是当前工程的project对象，所以在build.gradle直接使用ext=或者ext {}其实就是给project定义扩展属性

## Gradle 插件

### 什么是Gradle插件

- Gradle插件提供给Gradle构建工具，在编译时使用的依赖项，插件的本质就是对公用的构建业务进行打包，以提供复用，。
- Gradle插件分为：脚本插件和二进制插件（实现Plugin的类）
- Gradle插件通过apply方法引入工程

### 使用插件和脚本依赖项的区别

- 插件实现了一系列的任务，并且进行了组装，按照提供的API就可以直接使用
- 而Gradle脚本依赖项，是提供实现的任务封装，需要自行组装。或者使用到的一些具体业务的封装

### 自定义插件

- 实现gradle脚本，apply到指定工程。
- 实现自定义的插件类，继承Plugin，实现apply方法。
- 通过默认的buildsrc工程，直接实现插件的封装。在运行时，会自动打包成jar并进行依赖进来。



## Gradle依赖管理

### implementation

- Gradle会将依赖项添加到编译类路径，并将依赖打包到构建输出。不过，当您的模块配置implementation依赖时，会让Gradle了解您不希望该模块在编译时将该依赖项泄露给其他模块。也就是说其他模块只有在运行时才能使用该依赖项
- 使用此依赖配置代替api或complie可以显著缩短构建时间，因为这样可以减少构建系统需要重新编译的模块数。例如，如果implementation依赖想更改了其他API，Gradle只会重新编译该依赖项以及直接依赖于它的模块。大多数应用和测试模块都应使用此配置

### api

- Gradle只会将依赖项添加到编译路径（也就是说，不会将其添加到构建输出）。如果您创建Android模块时在编译期间需要相应的依赖项。但他在运行时可有可无，此配置会很有用。
- 此配置的行为类似于complie，但使用它时应格外小心们只能对您需要以传递方式导出到其他上游消费者的依赖项使用它。这是因为，如果api依赖项更改了其外部API，Gradle会在编译时重新编译所有有权访问该依赖项的模块。因此，拥有大量的API，依赖项回哦显著增加构建时间。除非要将依赖项的API公开给单独的模块，都则模块应该改用implementation依赖项



### compileOnly

- Gradle只会将依赖项添加到编译类路径（也就是说，不会将其添加到构建输出）。如果您创建Android模块时在编译时期间需要相应依赖项，但它在运行时可有可无，此配置会很有用。
- 如果您使用此配置，那么您的苦模块必须含有一个运行时条件，用于检查是否提供了相应的依赖项，然后是当地改变该模块行为，以使该模块在未提供相应依赖项的情况下仍可正常运行。这样做不会添加不重要的瞬时依赖项，因而有助于减小最终APK大小。此配置的行为类似于provided



### annotationProcessor

- 如果添加对作为注解处理器的库以来，您必须使用annotationProcessor配置将其添加到注解处理器的类路径。这是因为，使用此配置可以将编译类路径分开，从而提好构建性能，如果gradle在编译类路径上找到注释处理器，则会禁用避免编译功能，这样会对构建时间产生负面影响（Gradle 5.0 及更高版本会忽略在编译类路径上找到的注释处理器）。
- 如果jar文件包含以下文件，则Android Gradle插件会假定依赖项时注释处理器
- META-INFO/sercvices/javax.annotation.ptocessing.Processor。如果插件检查到编译类路径上包含注解处理器，则会产生构建错误。

