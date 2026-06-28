

## 设置用户名与邮箱

```
git config --global user.name "zhuangjx"
git config --global user.email "623155143@qq.com"
```

## 设置远程仓库

```
# 添加远程仓库，默认名称是 origin
git remote add origin https://github.com/ha321ha/zhuangjx.git

# 查看是否添加成功
git remote -v

# 修改
git remote set-url origin https://github.com/ha321ha/zhuangjx.git
```

## 提交

```
git commit -m "init"
```

## 分支

```
# 查看
git branch [-a]

# 创建并切换
git checkout -b [本地分支名] origin/master



```

## 本地分支与远程分支建立关联

```
# 全局
git config --global push.autoSetupRemote true

# 单个
git branch --set-upstream-to=origin/master master
```

## 强制推送覆盖

```

```


