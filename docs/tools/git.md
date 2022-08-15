# git



## git proxy

### 查看代理

```
git config --global --get http.proxy
git config --global --get https.proxy
```

### 设置代理

```
git config --global http.proxy http://127.0.0.1:19180
git config --global https.proxy http://127.0.0.1:19180
```

### 取消代理

```
git config --global --unset http.proxy
git config --global --unset https.proxy
```

