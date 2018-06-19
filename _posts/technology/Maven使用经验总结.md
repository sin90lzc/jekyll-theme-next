## 配置
### 系统配置

```sh
#设定Maven的安装目录
export M2_HOME=/*Maven安装目录*/
#设定Maven的执行命令搜索路径
export PATH=$PATH:$M2_HOME/bin
#设定jvm运行时内存分配，-Xms初始化内存，-Xmx最大可用内存
export MAVEN_OPTS=”-Xms=128m –Xmx=512m”
```

### Maven本身程序配置

#### 配置用户范围settings.xml

将全局配置文件`$M2_HOME/conf/settings.xml`文件复制至`~/.m2/settings.xml`。

常用的settings.xml文件已使用[configuration](https://github.com/sin90lzc/configuration)项目管理

## Maven的依赖

### 构件坐标

Maven中的构件坐标对应着仓库的文件路径，其规则如下：

```
groupId/artifactId/version/artifactId-version[-classifier].packaging
```

* groupId:以.为分隔后的单字作为目录，groupId不应该只表示哪个公司或组织，应该可以表示到某个项目。必填
* artifactId:名称应该是（项目名-模块）的形式。必填
* version:定义版本号。必填
* packaging:该元素定义打包方式，如jar或war。可选元素。
* classifier:该元素用来帮助定义构建输出的一些附属构件。这些附属构件由插件生成。可选元素。

> Note:在超级pom中定义的中央仓库地址http://repo.maven.apache.org/maven2

### 依赖配置

依赖配置中可以配置的元素如下代码所示：

```xml
<dependency>
	<groupId></groupId>
	<artifactId></artifactId>
	<version></version>
	<scope></scope>
	<type></type>
	<optional></optional>
	<systemPath></systemPath>
	<classifier></classifier>
	<exclusions>
		<exclusion></exclusion>
	</exclusions>
</dependency>
```

* groupId\artifactId\version：构件的基本坐标元素。
* type：对应项目打包类型packaging
* scope：用于控制多种classpath（编译主代码classpath，编译测试代码classpath，主程序运行* classpath，测试程序运行classpath）。可选值介绍如下：
	* compile：在编译主代码classpath，编译测试代码classpath，主程序运行classpath，测试程序运行classpath下都有效。
	* test：在编译测试代码classpath，测试程序运行classpath下有效。
	* provided：在编译主代码classpath，编译测试代码classpath，测试程序运行classpath下有效。如构件由容器提供时，那么在主程序运行classpath下就不需要该构件了。
	* runtime：在主程序运行classpath，测试程序运行classpath下有效。
	* system：与provided一致，但依赖文件路径由元素systemPath给出。
	* optional：可选依赖。可选依赖不会用于传递性依赖。如A->B->X(可选)，A->B->Y(可选)，那么X、Y并不会作为A的传递性依赖。
* systemPath：构件在本地路径地址，必须与scope:system一起使用。
* classifier：当pom一样时，用于区分version后面的部分，与坐标定义中的classifier对应。
* exclusions：排除一个或多个传递性依赖。

### 依赖范围对传递性依赖的影响

这个影响可以参考下图，图中最左边一行表示第一直接依赖范围，最上面一行表示第二直接依赖范围，中间的交叉单元格则表示传递性依赖范围。

![](https://sin90lzc.github.io/images/maven/scope.png)
 
由上图可以发现这样的规律：

* 当第二直接依赖的范围是compile的时候，传递性依赖的范围与第一直接依赖的范围一致；
* 当第二直接依赖的范围是test的时候，依赖不会得以传递；
* 当第二直接依赖的范围是provided的时候，只传递第一直接依赖范围也为provided的依赖，且传递性依赖的范围同样为provided；
* 当第二直接依赖的范围是runtime的时候，传递性依赖的范围与第一直接依赖的范围一致，但compile例外，此时传递性依赖的范围为runtime。

### 依赖调解原则

依赖调解第一原则：路径最近者优先。如A->B->C->X(1.0)、A->D->X(2.0)，则使用的是X(2.0)。

依赖调解第二原则：第一声明者优先。如A->B->Y(1.0)、A->C->Y(2.0)，则使用的是Y(1.0)。

### 优化依赖

* 查看当前项目已解析的依赖

```
mvn dependency:list
```

* 查看当前项目的依赖树

```
mvn dependency:tree
```

* 查看pom生效的配置

```
mvn help:effective-pom
```

> 分析当前项目的依赖，主要用于查看哪些依赖没有显式声明却被直接引用（Used undeclared dependencies）或依赖被显式声明却没有被使用(Unused declared dependencies)。

 
## Maven仓库

### 本地仓库

默认情况下，本地仓库的位置是在~/.m2/repository。但也可以在settings文件中修改这个默认仓库位置：

```xml
			<settings>
				<localRepository>/path/to/local/repo</localRepository>
			</settings>
```

> 可通过mvn install把项目打包成构件安装到本地仓库。

### 远程仓库

#### 中央仓库

中央仓库的配置可以查看超级pom。超级pom在`$M2_HOME/lib/maven-model-builder-XXX.jar`中找到。

#### 私服

私服一般架设在局域网中，私服的主要作用是提高组织内部项目构建的速度（因为不需要访问外网），在私服仓库中部署一些在外部仓库没有的构件供内部使用。

#### 外部远程仓库

外部远程仓库的配置可看下面的代码：

```xml
	<repositories>
		<repository>
			<id></id><!-- 仓库id -->
			<name></name><!-- 仓库名称 -->
			<url></url><!-- 仓库地址 -->
			<releases><!-- release与snapshots的配置类同 -->
				<enabled></enabled><!-- 是否提供release依赖的下载 -->
				<!-- 更新策略  -->
				<!-- 可选值：never-从不更新，always-每次构建都检查更新，interval:x-每隔x分钟检查更新  -->
				<updatePolicy></updatePolicy>
				<!-- 检查校验和文件策略，即检查校验和失败后怎么处理 -->
				<!-- warn-警告，但不影响构建，fail-直接构建失败，ignore-忽略校验和错误 -->
				<checksumPolicy></checksumPolicy>
			</releases>
			<snapshots></snapshots>
		</repository>
	</repositories>
```

#### 服务器认证配置<span id="auth"></span>

有时候在访问一些服务器时，需要进行认证的，这些认证信息一律配置在settings.xml中。可以参考项目[configuration](https://github.com/sin90lzc/configuration)中settings.xml文件的配置。如：

```xml
	<servers>
		<server>
			<id>releases</id>
			<username>tim</username>
			<password>751301435</password>
		</server>
	</servers>
```

#### 部署至远程服务器

部署至远程服务器的配置是为了可以在将项目打包并安装到本地仓库之余，还能将项目打包部署至远程服务器。这里的远程服务器在实际使用中一般指的是私服。下面是配置代码：

```xml
	<distributionManagement>
		<repository>
			<id></id>
			<name></name>
			<url></url>
		</repository>
		<snapshotRepository>
			<id></id>
			<name></name>
			<url></url>
		</snapshotRepository>
	</distributionManagement>
```

如果服务器需要认证，认证信息可参考[服务器认证配置](#auth)。当执行命令`mvn deploy`就可以部署至远程服务器。

#### 镜像

服务器镜像配置是配置在settings.xml文件的mirrors元素内。主要介绍一下mirror中的mirrorOf元素的配置，mirrorOf定义了镜像服务器可镜像的服务器有哪些，看看它的高级配置方法：

```xml
<mirrorOf>*</mirrorOf> <!--匹配所有远程仓库。-->
<mirrorOf>external:*</mirrorOf> <!--匹配所有远程仓库，使用localhost的除外，使用file:///的除外。-->
<mirrorOf>repo1,repo2</mirrorOf> <!--匹配仓库repo1和repo2，使用逗号分隔多个远程仓库-->
<mirrorOf>*,! repo1</mirrorOf> <!--匹配所有远程仓库，repo1除外，使用感叹号将仓库从匹配中排除。-->
```

## 插件与生命周期

### 三套生命周期

Maven拥有三套相互独立的生命周期，它们分别为clean、default和site。clean生命周期的目的是清理项目，default生命周期的目的是构建项目，而site生命周期的目的是建立项目站点。每套生命周期都由多个阶段(phase)组成，这些阶段是有顺序的，后面的阶段依赖于前面的阶段。这些阶段都可以与插件的目标绑定。

#### clean生命周期

1.	pre-clean：执行一些清理前需要完成的工作
2.	clean：清理上一次构建生成的文件。绑定的插件目标：maven-clean-plugin:clean
3.	post-clean：执行一些清理后需要完成的工作

#### default生命周期

1. validate 
2.	initialize 
3.	generate-sources：生成主代码
4.	process-sources：处理主代码，如变量替换
5.	generate-resources：生成主资源
6.	process-resources：处理主资源，如变量替换并复制至输出目录。
绑定插件目标maven-resources-plugin:resources
7.	compile：编译项目主代码，并把编译后的二进制文件放到输出目录
绑定插件目标maven-compile-plugin:compile
8.	process-classes：处理编译后的二进制文件
9.	generate-test-sources：生成测试代码
10.	process-test-sources：处理测试代码，如变量替换
绑定插件目标maven-resources-plugin:testResources
11.	generate-test-resources：生成测试资源
12.	process-test-resources：处理测试资源，如变量替换并复制至输出目录
13.	test-compile：编译测试代码，并把编译后的二进制文件放到输出目录
14.	process-test-classes：处理编译后的测试代码
15.	test：使用单元测试框架运行测试，测试代码不会被打包或部署。
绑定插件目标maven-surefire-plugin:test
16.	prepare-package：预打包阶段
17.	package：打包阶段。maven-jar-plugin:jar
18.	pre-integration-test：预集成测试阶段
19.	integration-test：集成测试阶段
20.	post-integration-test：后集成测试阶段
21.	verify：验证阶段
22.	install：安装阶段，将包安装到Maven本地仓库，供本地其他Maven项目使用。
绑定插件目标maven-install-plugin:install
23.	deploy：部署阶段，把包发布至远程仓库，供其他开发人员和Maven项目使用。
绑定插件目标maven-deploy-plugin:deploy

#### site生命周期

1. pre-site：执行一些在生成项目站点之前需要完成的工作。
2. site：生成项目站点文档。maven-site-plugin:site
3. post-site：执行一些在生成项目站点之后需要完成的工作。
4. site-deply：将生成的项目站点发布到服务器上。maven-site-plugin:deploy

### 插件目标与生命周期的绑定

生命周期阶段与插件目标是相互绑定的，因此在执行命令`mvn lifetime-phase`时，实际执行的是插件的目标。

下面是插件目标与生命周期阶段绑定的代码示例：

```xml
	<build>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-source-plugin</artifactId>
				<version>2.1.1</version>
				<executions>
					<execution>
						<id>attach-sources</id>
						<phase>verify</phase><!-- 绑定的生命周期阶段 -->
						<goals>
							<goal>jar-no-fork</goal><!-- 插件目标 -->
						</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>
```

有些插件目标在编写时已经默认与某个生命周期阶段绑定的了，可以通过下面的命令查看某个插件的信息：

```
mvn help:describe –Dplugin=org.apache.maven.plugins:maven-source-plugin:2.1.1 –Ddetail
```

在命令行中，插件目标的完整表示方式是：`groupId:artifactId:version:goal`

> 当多个插件目标被绑定到同一个生命周期阶段时，按插件声明的先后顺序执行。

### 插件配置

#### 命令行插件配置

例如，maven-surefire-plugin提供了一个maven.test.skip参数，当其值为true的时候，就会跳过执行测试。于是，在运行命令的时候，加上如下-D参数就能跳过测试：

```sh
$ mvn install –Dmaven.test.skip=true
```

#### POM中插件全局配置

不多说，看配置代码：

```xml
<build>
	<plugins>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-compiler-plugin</artifactId>
			<version>2.3.2</version>
			<configuration><!-- 插件的全局配置 -->
				<source>1.6</source>
				<target>1.6</target>
			</configuration>
		</plugin>
	</plugins>
</build>
```

> 如果插件不指定版本号，会使用最新稳定版本。

#### 插件按生命周期阶段划分配置

也不多说，看配置代码吧：

```xml
<build>
	<plugins>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-antrun-plugin</artifactId>
			<version>1.3</version>
			<executions>
				<execution>
					<id>ant-validate</id>
					<phase>validate</phase>
					<goals>
						<goal>run</goal>
					</goals>
					<configuration><!-- 这里的configuration在execution下面，按执行阶段划分配置  -->
						<tasks>
							<echo>I'm bound to validate phase.</echo>
						</tasks>
					</configuration>
				</execution>
				<execution>
					<id>ant-verify</id>
					<phase>verify</phase>
					<goals>
						<goal>run</goal>
					</goals>
					<configuration>
						<tasks>
							<echo>I'm bound to verify phase.</echo>
						</tasks>
					</configuration>
				</execution>
			</executions>
		</plugin>
	</plugins>
</build>
```

### 获取插件信息

#### 在线插件信息

有两个主要插件的网站提供了插件的说明文档：

```
http://maven.apache.org/plugins/index.html
http://mojo.codehaus.org/plugins.html
```

> 在这里需要注意一下，文档中都会提供插件目标的配置参数，这些参数可以直接在pom中设置，而不能用于命令行-D选项来定义参数，-D选项后面的参数名对应的是文档中的Expression。

#### 使用maven-help-plugin描述插件

不多说，看下面命令行：

```sh
$ mvn help:describe –Dplugin=org.apache.maven.plugins:maven-compiler-plugin:2.1 –Ddetail
```

### maven命令行的使用

Maven命令行的语法如下：

```
usage:mvn [options] [<goals>] [phases]
goals：指的是一个或多个插件目标，它的完整格式groupId:artifactId:version:goal
phases：指的是一个或多个生命周期阶段。
```

这里需要注意的是有些goals可以是这样的简化格式：prefix:goal。这个prefix是什么呢？
由于maven的插件大部分的groupId都是为`org.apache.maven.plugins`或`org.codehaus.mojo`，maven会把这两个groupId作为默认的groupId，因此在引用这些groupId的插件时，可以省略groupId。

也可以在settings.xml文件中添加更多的插件默认groupId:

```xml
<pluginGroups>
		<pluginGroup>com.sin90lzc</pluginGroup>
</pluginGroups>
```

在这些默认的groupId对应的本地仓库的groupId/maven-metadata.xml定义了插件的prefix。

```xml
<plugin>
<name>Maven Help Plugin</name>
<prefix>help</prefix>
<artifactId>maven-help-plugin</artifactId>
</plugin>
```

这就是为什么在命令行调用插件目标时，可以使用简写的prefix代替插件的原因。

## 聚合与继承

### 聚合

聚合pom的作用是在项目构建的时候，把多个模块联合起来，按反应堆顺序一次性完成项目构建任务。聚合pom模块应该除了pom.xml之外不包含其他任何内容。

```xml
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.sin90lzc.pom</groupId>
	<artifactId>aggregator-pom</artifactId>
	<version>2.0</version>
	<packaging>pom</packaging>
	<name>聚合pom</name>
	<modules>
		<!-- account-email模块在aggregator-pom模块的子目录 -->
		<module>account-email</module>
		<!-- account-persist与aggregator-pom是平行目录结构 -->
		<module>../account-persist</module>
	</modules>
```

### 继承

在maven中，pom是可以继承的，继承的意义是为了减少重复的配置，达到一处配置多处使用的目的。父pom的配置与一般pom配置没有太大的区别，除了packaging要配置为pom。

```xml
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.sin90lzc.pom</groupId>
	<artifactId>parent-pom</artifactId>
	<version>2.0</version>
	<packaging>pom</packaging>
```

其实，个人认为最好的实践是父pom与聚合pom都写在同一个pom.xml文件中，这个pom.xml在项目的根目录下，子模块则分配在子目录下。

下面的配置展示了子pom如何继承父pom的配置

```xml
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>com.sin90lzc.pom</groupId>
		<artifactId>parent-pom</artifactId>
		<version>2.0</version>
		<relativePath>../parent-pom/pom.xml</relativePath>
	</parent>
	<groupId>com.sin90lzc.training</groupId>
	<artifactId>training-mybatis</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>
```

> 必须在聚合pom中的modules元素中需要添加父pom模块。

### 可继承的pom元素

* groupId:项目组ID
* version:项目版本
* description:项目的描述信息
* organization:项目的组织信息
* inceptionYear:项目的创始年份
* url：项目的url地址
* developers: 项目的开发者信息
* contributors: 项目的贡献者信息
* distributionManagement:项目的部署配置
* issueManagement:项目的缺陷跟踪系统信息
* ciManagement:项目的持续集成系统信息
* scm:项目的版本控制系统信息
* mailingList:项目的邮件列表信息
* properties:自定义的Maven属性
* dependencies:项目的依赖配置
* dependencyManagement：项目的依赖管理配置
* repositories:项目的仓库配置
* build:包括项目的源码目录配置、输出目录配置、插件配置、插件管理配置等。
* reporting:包括项目的报告输出目录配置、报告插件配置等。

### 依赖继承管理配置

依赖继承管理的配置由元素dependencyManagement来定义。这个元素一般配置在父pom中。在这个元素下配置的依赖并不会导致子pom引入这些依赖，它的作用是为项目定义统一的依赖规范（依赖的版本号/范围），那么当子pom要引入这些依赖时，只需要在dependences中配置groupId和artifactId，版本号及范围则继承自dependencyManagement的配置。

### 插件继承管理配置

与dependencyManagement(5.4)类似，在<build>元素下有一个元素<pluginManagement>用于插件配置的继承管理。它的作用与dependencyManagement是一样的，因此，这里就不说了。
 
## 私服创建

有三种专业的Maven仓库管理软件可以建立私服：Apache的Archiva，JFrog的Artifactory和Sonatype的Nexus。Archiva是开源的，而Artifactory和Nexus的核心是开源的。

### Nexus的安装

下载地址：http://www.sonatype.org/nexus/

解压后，目录nexus-webapp-1.7.2/bin/jsw/windows-x86-32/下保存了windows的启动脚本：

* nexus.bat：启动脚本
* Installnexus.bat：将Nexus安装成Windows服务
* Uninstallnexus.bat：卸载Nexus Windows服务。
* Startnexus.bat：启动Nexus Windows服务
* Stopnexus.bat：停止Nexus Windows服务
* Pausenexus.bat：暂停Nexus Windows服务
* Resumenexus.bat：恢复暂停的Nexus Windows服务

目录nexus-webapp-1.7.2/bin/jsw/linux-x86-32/下保存了linux的启动脚本：

* ./nexus console：用控制台启动
* ./nexus start:在后台启动Nexus服务
* ./nexus stop:停止后台的Nexus服务
* ./nexus status:查看后台Nexus服务的状态
* ./nexus restart:重新启动后台的Nexus服务

Nexus的配置文件是nexus-webapp-1.7.2/conf/plexus.properties。

Nexus中默认的管理员帐号密码：admin/admin123

 
## 单元测试

在maven中，用于单元测试的插件是maven-surefire-plugin，它支持java中主流的单元测试框架JUnit(http://www.junit.org)和TestNG(http://testng.org)。这个插件是在maven执行到特定生命周期时调用Junit或TestNG的测试用例。

### 跳过单元测试

属性skipTests（命令行表达式skipTests）：跳过测试，但依然会对测试代码进行编译

```
$ mvn package -DskipTests
```

属性skip（命令行表达式maven.test.skip）：跳过测试，而且不会对测试代码进行编译

```
$ mvn package –Dmaven.test.skip=true
```

### 指定单元测试用例

只执行RandomTest测试用例

```
$ mvn test –Dtest=RandomTest
```

执行所有符合Random*Test的测试用例

```
$ mvn test –Dtest=Random*Test
```

执行AppTest及所有符合Random*Test的测试用例

```
$ mvn test –Dtest=AppTest,Random*Test
```

如果不存在任何测试用例也不会报错

```
$ mvn test –DfailIfNoTests=false
```

maven-surefire-plugin插件中有两个配置属性，用于包括或排除测试用例

```xml
<configuration>
	<excludes>
		<exclude>**/*ServiceTest.java</exclude>
		<exclude>**/NonTest.java</exclude>
	</excludes>
	<includes>
		<include>**/*Tests.java</include>
	</includes>
</configuration>
```

### 测试报告

#### 基本测试报告

默认情况下，运行完测试后会在target/surefire-reports下生成两种格式的报告，一种是简单文本格式，另一种是Junit兼容的xml文件（可以被其他应用解析，如hudson，eclipse）

#### 测试覆盖率报告

测试覆盖率是衡量项目代码质量的一个重要的参考指标。Cobertura是一个优秀的开源测试覆盖率统计工具（详见http://cobertura.sourceforge.net), Maven通过插件cobertura-maven-plugin与之集成。运行如下命令生成测试覆盖率报告：

```
$ mvn cobertura:cobertura
```

生成的报告保存在target/site/cobertura/下，打开index.html文件可查看报告。

### 重用测试代码

Maven在默认情况下只对主代码、主资源进行打包，而不会对测试代码、测试资源进行打包。有些时候可能会对测试代码进行重用，这时候就要对测试代码进行打包并安装到本地仓库或远程仓库提供给其他项目引用。通过下面的配置可以修改这种默认行为：

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-jar-plugin</artifactId>
	<version>2.2</version>
	<executions>
		<execution>
			<goals>
				<goal>test-jar</goal><!-- 这里默认的目标是jar -->
			</goals>
		</execution>
	</executions>
</plugin>
```

而在其他项目引入时，注意构件类型为test-jar

```xml
		<dependency>
			<groupId>org.slf4j</groupId>
			<artifactId>slf4j-simple</artifactId>
			<version>1.0</version>
			<type>test-jar</type><!-- 注意是test-jar -->
			<scope>test</scope>
		</dependency>
```

### Maven与TestNG

TestNG是Java社区中除了JUnit之外另一个流行的单元测试框架。TestNG在JUnit的基础上增加了很多特性，其站点是http://testng.org/.添加TestNG依赖

```xml
<dependency>
	<groupId>org.testng</groupId>
	<artifactId>testng</artifactId>
	<version>5.9</version>
	<scope>test</scope>
	<classifier>jdk15</classifier>
</dependency>
```

下面是JUnit和TestNG的常用类库对应关系

|JUnit类|TestNG类|作用|
|-----|-----|-----|
|org.junit.Test|org.testng.annotations.Test|标注方法为测试方法|
|org.junit.Assert|org.testng.Assert|检查测试结果|
|org.junit.Before|org.testng.annotations.BeforeMethod|标注方法在每个测试方法之前运行|
|org.junit.After|org.testng.annotations.AfterMethod|标注方法在每个测试方法之后运行|
|org.junit.BeforeClass|org.testng.annotations.BeforeClass|标注方法在所有测试方法之前运行|
|org.junit.AfterClass|org.testng.annotations.AfterClass|标注方法在所有测试方法之后运行|

TestNG允许用户使用一个名为testng.xml的文件来配置想要运行的测试集合。如在类路径上添加testng.xml文件，配置只运行RandomGeneratorTest

```xml
<?xml version="1.0" encoding="UTF-8"?>
<suite name="Suite1" verbose="1">
	<test name="Regression1">
		<classes>
			<class name="com.juvenxu.mvnbook.account.captcha.RandomGeneratorTest" />
		</classes>
	</test>
</suite>
```

同时再配置maven-surefire-plugin使用该testng.xml，如：

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-surefire-plugin</artifactId>
	<version>2.5</version>
	<configuration>
		<suiteXmlFiles>
			<suiteXmlFile>testng.xml</suiteXmlFile>
		</suiteXmlFiles>
	</configuration>
</plugin>
```

TestNG较JUnit的一大优势在于它支持测试组的概念。如可以在方法级别声明测试组：

```java
@Test(groups={"util","medium"})
```

然后可以在pom中配置运行一个或多个测试组：

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-surefire-plugin</artifactId>
	<version>2.5</version>
	<configuration>
		<groups>util,medium</groups>
	</configuration>
</plugin>
```

## 持续集成

一次完整的集成往往会包括以下6个步骤：

1. 持续编译：所有正式的源代码都应该提交到源码控制系统中（如subversion），持续集成服务器按一定频率检查源码控制系统，如果有新的代码，就触发一次集成，旧的已编译的字节码应当全部清除，然后服务器编译所有最新的源码。
2. 持续数据库集成：在很多项目中，源代码不仅仅指Java代码，还包括了数据库SQL脚本，如果单独管理它们，很容易造成与项目其他代码的不一致，并造成混乱。持续集成也应该包括数据库的集成，每次发现新的SQL脚本，就应该清理集成环境的数据库，重新创建表结构，并填入预备的数据。这样就能随时发现脚本的错误，此外，基于这些脚本的测试还能进一步发现其他相关的问题。
3. 持续测试：有了Junit之类的框架，自动化测试就成了可能。编写优良的单元测试并不容易，好的单元测试必须是自动化的、可重复执行的、不依赖于环境的，并且能够自我检查的。除了单元测试，有些项目还会包含一些依赖外部环境的集成测试。所有这些测试都应该在每次集成的时候运行，并且在发生问题的时候能产生具体报告。
4. 持续审查：诸如Checkstyle和PMD之类的工具能够帮我们发现代码中的坏味道(Bad Smell)，持续集成可以使用这些工具生成各类报告，如测试覆盖率报告、Checkstyle、PMD报告等。这些报告的生成频率可以低一些，如每日生成一次，当审查发现问题的时候，可以给开发人员反馈警告信息。
5. 持续部署：有些错误只有在部署后才能被发现，它们往往是具体容器或者环境相关的，自动化部署能够帮助我们尽快发现这类问题。
6. 持续反馈：持续集成的最后一步的反馈，通常是一封电子邮件。在重要的时候将正确的信息发送给正确的人。如果开发者一直受到与自己无关的持续集成报告，他慢慢地就会忽略这些报告。基本的规则是：将集成失败报告发送给这次集成相关的代码提交者，项目经理应该收到所有失败报告。

### Hudson安装

下载地址：http://hudson-ci.org

这里下载之后是一个war文件，后面的就不用说了。

## Maven对Web的支持

一个典型的Web项目在Maven下的目录结构如下：

![](https://sin90lzc.github.io/images/maven/web.png)
 
从图中可以看出Maven Web的目录结构与一般的Maven目录结构的区别是：

在src/main/中多了一个webapp的目录，该目录下存放的就是WEB-INF的内容。

Maven Web除了目录结构不一样外，还需要把pom.xml的<packaging>配置为war。大家都知道web的打包方式是war。

### 如何使用jetty-maven-plugin进行调试

传统的Web测试方法要求我们编译、测试、打包及部署，这往往会消耗数10秒至数分钟的时间，jetty-maven-plugin能够帮助我们节省时间，它能够周期性地检查项目内容，发现变更后自动更新到内置的Jetty Web容器中，换句话说，就是能帮我们省去了打包及部署的时间。

要使用jetty-maven-plugin，只需要在pom中稍加配置就可以了。如：

```
<plugin>
	<groupId>org.mortbay.jetty</groupId>
	<artifactId>jetty-maven-plugin</artifactId>
	<version>7.1.6.v20100715</version>
	<configuration>
	<!-- 插件扫描项目的时间间隔 -->
		<scanInterwebAvalSeconds>10</scanIntervalSeconds>
		<webAppConfig>
		<!-- web应用访问的contextpath。用户便可以通过http://hostname:port/test/ -->
			<contextPath>/test</contextPath>
		</webAppConfig>
	</configuration>
</plugin>
```

由于默认情况下，只有org.apache.maven.plugins和org.codehaus.mojo两个groupId下的插件才支持简化的命令行调用，如mvn help:system，但jetty-maven-plugin不属于默认情况，为了能简化jetty-maven-plugin的命令，还需要配置settings.xml：

```
<settings>
	<pluginGroups>
		<pluginGroup>org.mortbay.jetty</pluginGroup>
	</pluginGroups>
</settings>
```

现在就可以使用下面命令启动Jetty，并默认监听本地的8080端口，并将当前项目部署到容器中，同时扫描代码改动：

```sh
mvn jetty:run
```

如果想要使用其他端口，可以添加jetty.port参数。如：

```sh
mvn jetty:run -Djetty.port=9999
```

如果想要进一步了解jetty-maven-plugin插件，可以访问[Jetty_Maven_Plugin](http://wiki.eclipse.org/Jetty/Feature/Jetty_Maven_Plugin)

### 使用Cargo实现自动部署

略
 
## Maven灵活构建

Maven内置了三大特性：属性、Profile和资源过滤来支持构建的灵活性。

### Maven属性

事实上有六种类型的Maven属性：

* 内置属性：主要有两个常用内置属性——${basedir}表示项目根目录，即包含pom.xml文件的目录;${version}表示项目版本。
* POM属性：pom中对应元素的值。例如${project.artifactId}对应了`<project><artifactId>`元素的值。具体有哪些POM属性可以用，可以查看本页末的附件——超级POM
* 自定义属性：在pom中<properties>元素下自定义的Maven属性。例如

```xml
<project>
	<properties>
		<my.prop>hello</my.prop>
	</properties>
</project>
```

* Settings属性：与POM属性同理。如${settings.localRepository}指向用户本地仓库的地址。
* Java系统属性：所有Java系统属性都可以使用Maven属性引用，例如${user.home}指向了用户目录。可以通过命令行mvn help:system查看所有的Java系统属性
* 环境变量属性：所有环境变量都可以使用以env.开头的Maven属性引用。例如${env.JAVA_HOME}指代了JAVA_HOME环境变量的值。也可以通过命令行mvn help:system查看所有环境变量。


### 资源过滤

默认情况下，Maven属性只有在POM中才会被解析。资源过滤就是指让Maven属性在资源文件(src/main/resources、src/test/resources)中也能被解析。

在POM中添加下面的配置便可以开启资源过滤

```xml
<build>
	<resources>
		<resource>
			<directory>${project.basedir}/src/main/resources</directory>
			<filtering>true</filtering>
		</resource>
	</resources>
	<testResources>
		<testResource>
			<directory>${project.basedir}/src/test/resources</directory>
			<filtering>true</filtering>
		</testResource>
	</testResources>
</build>
```

从上面的配置中可以看出，我们其实可以配置多个主资源目录和多个测试资源目录。

Maven除了可以对主资源目录、测试资源目录过滤外，还能对Web项目的资源目录(如css、js目录)进行过滤。这时需要对maven-war-plugin插件进行配置

```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-war-plugin</artifactId>
	<version>2.1-beta-1</version>
	<configuration>
		<webResources>
			<resource>
				<filtering>true</filtering>
				<directory>src/main/webapp</directory>
				<includes>
					<include>**/*.css</include>
					<include>**/*.js</include>
				</includes>
			</resource>
		</webResources>
	</configuration>
</plugin>
```

### Maven Profile

每个Profile可以看作是POM的一部分配置，我们可以根据不同的环境应用不同的Profile，从而达到不同环境使用不同的POM配置的目的。

profile可以声明在以下这三个文件中：

* pom.xml：很显然，这里声明的profile只对当前项目有效
* 用户settings.xml：.m2/settings.xml中的profile对该用户的Maven项目有效
* 全局settings.xml：conf/settings.xml，对本机上所有Maven项目有效非常值得注意的一点是，profile在pom.xml中可声明的元素在settings.xml中可声明的元素是不一样的：
* profile在pom.xml中可声明的元素：

```xml
<project>
	<repositories></repositories>
	<pluginRepositories></pluginRepositories>
	<distributionManagement></distributionManagement>
	<dependencies></dependencies>
	<dependencyManagement></dependencyManagement>
	<modules></modules>
	<properties></properties>
	<reporting></reporting>
	<build>
		<plugins></plugins>
		<defaultGoal></defaultGoal>
		<resources></resources>
		<testResources></testResources>
		<finalName></finalName>
	</build>
</project>
```

* profile在settings.xml中可声明的元素：

```xml
<project>
	<repositories></repositories>
	<pluginRepositories></pluginRepositories>
	<properties></properties>
</project>
```

有多种激活Profile的方式：

* 命令行方式激活，如有两个profile id为devx和devy的profile：

```
mvn clean install  -Pdevx,devy
```

* settings文件显式激活

```xml
<settings>
	...
	<activeProfiles>
		<activeProfile>devx</activeProfile>
		<activeProfile>devy</activeProfile>
	</activeProfiles>
	...
</settings>
```

* 系统属性激活，用户可以配置当某系统属性存在或其值等于期望值时激活profile，如：

```xml
<profiles>
	<profile>
		<activation>
			<property>
				<name>actProp</name>
				<value>x</value>
			</property>
		</activation>
	</profile>
</profiles>
```

不要忘了，可以在命令行声明系统属性。如：

```
mvn clean install -DactProp=x
```

* 操作系统环境激活，如

```xml
<profiles>
	<profile>
		<activation>
			<os>
				<name>Windows XP</name>
				<family>Windows</family>
				<arch>x86</arch>
				<version>5.1.2600</version>
			</os>
		</activation>
	</profile>
</profiles>
```

这里的family值包括Window、UNIX和Mac等，而其他几项对应系统属性的os.name、os.arch、os.version

* 文件存在与否激活，Maven能根据项目中某个文件存在与否来决定是否激活profile

```xml
<profiles>
	<profile>
		<activation>
			<file>
				<missing>x.properties</missing>
				<exists>y.properties</exists>
			</file>
		</activation>
	</profile>
</profiles>
```

插件maven-help-plugin提供了一个目标帮助用户了解当前激活的profile：

```
mvn help:active-profiles
```

另外还有一个目标来列出当前所有的profile：

```
mvn help:all-profiles
```


