# 实战

## 一、创建插件

### 第一步 新建立一个Model

### 第二步骤 修改build.gradle 让其成为插件项目

```groovy
apply plugin: 'groovy'
apply plugin: 'maven'

repositories {
    mavenCentral()
}


dependencies {
    implementation gradleApi()
    implementation localGroovy()
    compile 'com.android.tools.build:gradle:3.3.2'
    compile 'org.javassist:javassist:3.21.0-GA'
    compile 'org.ow2.asm:asm:6.0'
}


uploadArchives {
    // 打成一个jar
    repositories.mavenDeployer{
        pom{
            groupId  = 'com.meituan.javassit'
            artifactId = 'plugin'
            version = '1.0.0'
        }
        repository(url:uri('../repo'))
    }

```

### 第三步 mian下groovy目录

### 第四步 创建插件类

```groovy
public class JavaSisitPlugin implements Plugin<Project> {
    @Override
    public void apply(Project project) {
        println "JavaSisitPlugin apply"

        def transform = new ModifyTransform()
        transform.project = project
        // 注册Transform
        project.android.registerTransform(transform)
    }
}
```

### 第五步 创建对应的声明配置resources/META-INF/gradle-plugins/javassit.properties

javassit.properties的javassit决定了插件的名称

## 二、javasist

```groovy
public class ModifyTransform extends Transform {
    Project project;
    def pool = ClassPool.default;

    @Override
    void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        super.transform(transformInvocation)


        project.android.bootClasspath.each {
            pool.appendClassPath(it.absolutePath)

        }

        // 拿到输入
//
        transformInvocation.inputs.each {
            it.directoryInputs.each {
                def absolutePath = it.file.absolutePath;
                pool.insertClassPath(absolutePath)
                findTarget(it.file, absolutePath)
                File outputDirectory = transformInvocation.outputProvider
                        .getContentLocation(it.name, it.contentTypes, it.scopes, Format.DIRECTORY)
                FileUtils.copyDirectory(it.file, outputDirectory)
            }
            it.jarInputs.each {
                File outputJarFile = transformInvocation.outputProvider
                        .getContentLocation(it.name, it.contentTypes, it.scopes, Format.JAR);
                FileUtils.copyFile(it.file, outputJarFile)
            }
        }
    }

    private void findTarget(File file, String name) {
        if (file.isDirectory()) {
            file.listFiles().each {
                findTarget(it, name)
            }
        } else {
            def filePath = file.path;
            if (filePath.endsWith(".class")) {

                modify(filePath, name)
            }
        }
    }

    private void modify(String filePath, String absolutePath) {
        if (filePath.contains('R$') || filePath.contains("R.class") || filePath.contains("BuildConfig.class")) {
            return
        }
        println "=============找到class============" + filePath

        def className = filePath.replace(absolutePath, "").replace("\\", ".").replace("/", ".").replace(".class", "")
        className = className.substring(1, className.length())
        println "=============className============" + className
        CtClass ctClass = pool.get(className)

        addCode(ctClass, absolutePath)
    }

    private void addCode(CtClass ctClass, String absolutePath) {
        ctClass.defrost()
//        ctClass.addField()
        ctClass.getMethods().each {
            it.insertBefore("if(true){")
            it.insertAfter("}")
            it.inse

        }
        ctClass.writeFile(absolutePath)
        ctClass.detach()

    }

    @Override
    String getName() {
        return "renyushuang"
    }

    @Override
    Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS
    }

    @Override
    Set<? super QualifiedContent.Scope> getScopes() {
        return TransformManager.SCOPE_FULL_PROJECT
    }

    @Override
    boolean isIncremental() {
        return false
    }

}
```

### Transform的一些关键方法

```groovy
public class ModifyTransform extends Transform {
   
    @Override
    void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        super.transform(transformInvocation) 
      	// 节点方法，在该处处理逻辑
    }

    @Override
    String getName() {
    		// 决定了Transform的目录名称
        return "renyushuang"
    }

    @Override
    Set<QualifiedContent.ContentType> getInputTypes() {
    		// 输入类型，目前输入的类型是class，还可以搞资源
        return TransformManager.CONTENT_CLASS
    }

    @Override
    Set<? super QualifiedContent.Scope> getScopes() {
      	// Transform使用的范围
        return TransformManager.SCOPE_FULL_PROJECT
    }

    @Override
    boolean isIncremental() {
        return false
    }

}
```

### 第一步 

```groovy
    @Override
    void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        super.transform(transformInvocation)


        project.android.bootClasspath.each {
            pool.appendClassPath(it.absolutePath)
        }

        // 拿到输入
        transformInvocation.inputs.each {
            it.directoryInputs.each {
              	// 拿到目录下的
                def absolutePath = it.file.absolutePath;
                pool.insertClassPath(absolutePath)
                findTarget(it.file, absolutePath)
                File outputDirectory = transformInvocation.outputProvider
                        .getContentLocation(it.name, it.contentTypes, it.scopes, Format.DIRECTORY)
              	// 拷贝文件到Transform问题
                FileUtils.copyDirectory(it.file, outputDirectory)
            }
            it.jarInputs.each {
               	// 拿到jar
                File outputJarFile = transformInvocation.outputProvider
                        .getContentLocation(it.name, it.contentTypes, it.scopes, Format.JAR);
              	// 拷贝文件到Transform问题
                FileUtils.copyFile(it.file, outputJarFile)
            }
        }
    }
```

### 遍历到.class

```groovy
private void findTarget(File file, String name) {
    if (file.isDirectory()) {
        file.listFiles().each {
            findTarget(it, name)
        }
    } else {
        def filePath = file.path;
        if (filePath.endsWith(".class")) {

            modify(filePath, name)
        }
    }
}
```



### 修改Class

```groovy
    private void modify(String filePath, String absolutePath) {
        if (filePath.contains('R$') || filePath.contains("R.class") || filePath.contains("BuildConfig.class")) {
            return
        }
        println "=============找到class============" + filePath
				// 找到类的全路径
        def className = filePath.replace(absolutePath, "").replace("\\", ".").replace("/", ".").replace(".class", "")
        className = className.substring(1, className.length())
        println "=============className============" + className
        CtClass ctClass = pool.get(className)
				
     		// 增加方法
        addCode(ctClass, absolutePath)
    }

    private void addCode(CtClass ctClass, String absolutePath) {
        ctClass.defrost()
//        ctClass.addField()
        ctClass.getMethods().each {
            it.insertBefore("if(true){}")
        }
        ctClass.writeFile(absolutePath)
        ctClass.detach()

    }
```