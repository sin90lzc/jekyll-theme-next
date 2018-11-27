# 创建maven archetype的一般过程

1. 创建一个一般的maven项目
2. 在该maven项目下执行`mvn archetype:create-from-project`（该命令使用的是插件`Maven Archetype Plugin`，执行完该命令后，会在项目的`target/generated-sources/archetype`下生成骨架的源文件。
3. 编辑骨架源文件下的`target/generated-sources/archetype/src/main/resources/META-INF/maven/archetype-metadata.xml`文件，按需排除或添加需要的骨架文件。
4. 此时，项目的骨架源文件已经有了，就可以在`target/generated-sources/archetype`下执行`mvn install`或`mvn deploy`命令，打包骨架并安装到本地或发布到远程仓库。
5. 骨架打包安装完成之后，就可以使用下列命令应用项目骨架了：

```shell
    mvn archetype:generate \
    -DgroupId=cn.oigo.product \
    -DartifactId=oigo-product \
    -DarchetypeArtifactId=demo-archetype \
    -DarchetypeGroupId=cn.oigo.demo \
    -DarchetypeVersion=2.0 \
    -DinteractiveMode=false
```

# 自定义项目骨架文件内容

创建maven archetype的一般过程可以说是对maven项目的一份复制，但如果需要对复制后的项目文件进行修改时，就需要使用`src/main/resources/META-INF/archetype-post-generate.groovy`脚本对生成的项目骨架进行修改了(参考[maven官网](http://maven.apache.org/archetype/maven-archetype-plugin/advanced-usage.html)。在执行`mvn archetype:create-from-project`命令之后，`archetype-post-generate.groovy`会自动地生成到`target/generated-sources/archetype/src/main/resources/META-INF/archetype-post-generate.groovy`，下面是`archetype-post-generate.groovy`的示例：

```groovy
def moduleDir = new File(request.getOutputDirectory()+"/"+request.getArtifactId());
def appName=request.getArtifactId().split("-")[-1];

def bootstrapFile=new File(moduleDir,"app/src/main/resources/bootstrap.properties");

//println(bootstrapFile);
Properties props = new Properties()
props.load(bootstrapFile.newDataInputStream())
props.setProperty("spring.application.name",appName);
props.store(bootstrapFile.newWriter(), null);

def localconfig=new File(moduleDir,"app/src/main/resources/local_config/application.yaml");

//println(localconfig);
localconfig.text=localconfig.text.replaceAll("security",appName);
```