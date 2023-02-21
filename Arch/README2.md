# pacman mirrorlist

镜像可能需要隔一段时间需要进行更新，除了手动修改文件`/etc/pacman.d/mirrorlist`外，还可以通过软件`reflector`进行自动更新，常用的更新命令为：

```shell
$ sudo reflector --verbose --contry China --age 12 --sort rate --save /etc/pacman.d/mirrorlist
```