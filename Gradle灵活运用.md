## Gradle灵活运用

##打出变体包的步骤

#### 1.配置签名、渠道（applicationId）

  选择app module，选择Signing选项卡，首先配置好了这两个变体版本的签名文件，文件名可任意取（见名知意）。 
![这里写图片描述](http://img.blog.csdn.net/20170327140441650?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYW5kcm9pZF9hcHBfMjAxMg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast) 
  签名属性字段描述：

- keyAlias//指定签名文件的别名
- keyPassword//指定签名文件的别名配对密码
- storeFile//签名文件路径
- storePassword//签名文件的密码

  切换到Flavors选项卡，配置变体包的相关属性，由于下一步创建资源文件夹时，需要与这里的文件名对应，所以此处的取名尽量简约。这里我创建两个命名dev和rea的Flavor，所有的Flavor都会复写defaultConfig中的属性，所以可以看到我并没有填写其中的一些属性。在这里非常关键的一点，是对不同的变体设置不同的applicationId，并选择对应的Signing Config。 
![这里写图片描述](http://img.blog.csdn.net/20170327140617386?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYW5kcm9pZF9hcHBfMjAxMg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 2.新建不同变体的sourceSet

  在修改完变体的配置文件后，我们还需要再项目的src文件夹下，新建以我们的Flavor名称命名的文件夹，并在这些文件夹下新建如main中相同的目录结构。 
![这里写图片描述](http://img.blog.csdn.net/20170327140759574?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYW5kcm9pZF9hcHBfMjAxMg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast) 
  我们正常编写项目都是写在main这个sourceSet下的，但是如果我们的项目的变体有不同的资源文件、Java文件时，我们就需要使用不同的sourceSet来区别开。 
  需要注意的是，Flavor下的资源文件会与main中的合并，如果存在重复，则Flavor中优先级高于main中。我们可以将不同变体中共用的资源存放在main中，只将不同的内容存放在flavor的sourceSet中。 
  如果不同变体有内容不同的Java文件则要注意，需要将这个Java文件放置到每个flavor 的sourceSet文件夹下，main中不可以有这个Java文件，如果main中也存在此文件，编译时会提示文件重复。比如说有两个变体，有着不同的TargetActivity.java，那么main中就不能有这个文件了，需要把这个Java文件放到各个flavor的sourceSet下，同时这个Java文件在sourceSet中要按照main中的包结构保存。

#### 3.关于配置不同的launcher图标和应用名

  通过之前的操作，已经可以有效区分变体包的applicationId、Signing Config和java文件，那么应用logo和name又该如何处理呢，这里有两种方法。

> 1）配置不同的资源：我们可以使用类似于处理java文件的方法，在不同的sourceSet中配置string.xml中app_name属性，在mipmap文件夹中放置对应的ic_app.png图标资源。不过当我们需要更换启动图和应用名而变体包又比较多的情况下，这种方法会比较麻烦，因为我们要到每一个sourceSet资源文件下去对应的修改，如果使用占位符的方式，那么会变得简单一些。 
> 2）使用占位符的方式：在AndroidManifest文件中使用占位符然后在flavor中直接配置，在flavor中使用的属性是manifestPlaceholderss（manifestPlaceholders =[NAME1:VALUE1]），而图标资源和应用名称我们都可以直接放置到main中。 
> 结果如下图： 
> ![这里写图片描述](http://img.blog.csdn.net/20170327141144269?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYW5kcm9pZF9hcHBfMjAxMg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 常用gradlew命令

#### 八个常用命令：

> - ./gradlew -v //查看gradle版本
> - ./gradlew assembleDebug //编译并打出Debug版本的包
> - ./gradlew assembleRelease //编译并打出Release版本的包
> - ./gradlew build //执行检查并编译打包，打出所有Release和Debug的包
> - ./gradlew clean //删除build目录，会把app下面的build目录删掉
> - ./gradlew installDebug //编译打包并安装Debug版本的包
> - ./gradlew uninstallDebug //卸载Debug版本的包
> - ./gradlew -info //使用-info查看任务详情