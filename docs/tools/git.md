# git



## git config

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

### 设置编辑器
```
git config -–global core.editor vim
```

## git commands

### 撤销操作
1. 撤销 `add` 操作:
    
    使用`reset`命令
    - 后面什么都不跟的，就是上一次add 里面的内容全部撤销
    - 后面跟文件名，就是对某个文件进行撤销
    ```
    git reset HEAD 
    git reset HEAD XXX 
    ```

    在 `HEAD` 后面加 `^` 或者 `~` 其实就是以 `HEAD` 为基准，来表示之前的版本，因为 `HEAD` 被认为是当前分支的最新版本，那么 `HEAD~` 和 `HEAD^` 都是指次新版本，也就是倒数第二个版本，`HEAD~~` 和 `HEAD^^` 都是指次次新版本，也就是倒数第三个版本，以此类推。

2. 撤销`commit` 操作(尚未push到remote): 
    ```
    git reset --soft HEAD^ 
    ```
    注意，仅仅是撤回commit操作，对文件的改动仍然保留。
    与`--soft`相对的`--hard` 将会撤销`commit`撤销`add`， 并**删除工作空间的改动代码**

3. 撤销`commit` 命令的注释:
    ```
    git commit --amend
    ```
4. 撤销`push` 操作:
    ```
    git reset --soft HEAD^ 
    git push --force
    ```
