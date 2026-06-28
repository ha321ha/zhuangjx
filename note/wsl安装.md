# 

## 指定目录安装

管理员权限执行

```
wsl --install -d Ubuntu-24.04 --location F:\wsl\ubuntu
```

## 开启镜像网络

 %UserProfile%\.wslconfig

```
  [wsl2]
  networkingMode=mirrored
  dnsTunneling=true      # DNS 走宿主隧道，解决某些网络下解析失败
  firewall=true          # 让 Hyper-V 防火墙规则也作用到 WSL
  autoProxy=true         # 自动继承 Windows 的代理设置
```

## 关闭

```
wsl --shutdown
```

## 开机自动启动

```
按 Win + R，输入 shell:startup 回车 → 打开启动文件夹，在里面新建一个快捷方式：
将现有开启的快捷方式复制进去。


关闭
删掉 shell:startup 启动文件夹里的那个快捷方式
```


