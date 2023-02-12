## 常用命令：
1. 查看所有虚拟环境
    ```bash
    conda info -e
    ```

2. 创建虚拟环境
    ```bash
    conda create -n <env name> python=<py version> [packages...]
    ```

3. 切换虚拟环境
    ```bash
    conda activate <env name>
    conda deactivate <env name>
    ```

4. 删除虚拟环境
    ```bash
    conda remove -n <env name> --all
    ```

5. 删除没用的包和缓存，释放磁盘空间

    ```bash
    conda clean --all
    ```

