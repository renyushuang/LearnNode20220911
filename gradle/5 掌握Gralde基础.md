## 环境配置

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

