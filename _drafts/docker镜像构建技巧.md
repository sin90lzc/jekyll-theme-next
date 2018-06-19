## 清理构建过程中多余的内容

### 对比两个镜像

可以使用google的[container-diff](https://github.com/GoogleContainerTools/container-diff)工具来分析两个镜像的差异，通过分析镜像差异来清理Dockerfile构建过程中多余的文件，以减少镜像体积。



### alpine

如果采用工alpine的apk安装工具，在安装完之后可清理索引缓存：

```sh
rm -rf /var/cache/apk
```


