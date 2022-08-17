- 查看环境变量

  ```powershell
  dir env:	 		# dir 是 Get-ChildItem 的alias
  $env:path			# 查看某一具体环境变量
  ```

  

- 给powershell设置全局代理

  ```powershell
  $env:HTTP_PROXY="http://127.0.0.1:19180"
  $env:HTTPS_PROXY="http://127.0.0.1:19180"
  ```

  vcpkg等工具使用cmake管理库，cmake使用Curl下载归档的库，而Curl使用这两个环境变量，因此设置这两个环境变量可以解决vcpkg下载慢问题

- 