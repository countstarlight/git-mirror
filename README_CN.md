# git-mirror

[English](README.md) | 简体中文

用于在Git仓库之间设置和**双向同步**来进行镜像的脚本，下面还介绍了单向镜像

## `scripts/setup-git-mirror.sh`

用于设置一个远程 Git 仓库镜像

**警告： 镜像也许会破坏仓库中的版本记录**

`git clone --mirror` 意味着 `--bare` 和对源远程仓库(remote)进行设置，使得运行
`git fetch` 将直接拉取修改到本地分支而不进行合并(merge)。这将进行强制更新，所以如果本地仓库和远程仓库的修改不一致，将会丢失本地仓库的修改。

`git push --mirror` 类似：不会只推送一个分支，它确保远程仓库和本地仓库的所有引用(分支，标签等)都一致，这表示也会进行强制更新

**强烈建议在远程镜像中保护重要的分支和标签(tags)，用Git更新钩子来阻止意外的修改，示例在`scripts/git-update-hook-for-protecting-branches-and-tags`**，将它复制到 `hooks/update` ，用命令 `chmod 755 hooks/update` 给它在远程仓库中执行的权限

## `scripts/synchronize-git-mirror.sh`

用于同步之前用 `scripts/setup-git-mirror.sh` 设置的镜像

这个脚本会被镜像仓库的 `post-receive` 钩子和 `crontab` 调用

有加锁操作使得两边同时调用时不会导致冲突

## 日志

同步日志被保存到文件 `scripts/synchronize-git-mirror.log*`.

## 配置文件

配置文件在 `scripts/synchronize-git-mirror.config`，注释介绍了每个配置项的用途

## 单向镜像

简单的单向镜像设置：

    +------------+                +------------+                +------------+
    |            |                |            |                |            |
    |   源仓库    <-- 拉取(pull) --+   镜像仓库   +-- 推送(push) --->  目标仓库   |
    |            |                |            |                |            |
    +------------+                +------------+                +------------+

步骤：

1. 添加系统用户，生成SSH密钥(key)，将公钥添加到源仓库：

    ```shell
    sudo adduser --system gitmirror
    sudo -u gitmirror ssh-keygen
    sudo cat /home/gitmirror/.ssh/id_rsa.pub
    ```
  
2. 测试clone是否正常：

    ```shell
    cd /tmp && sudo -u gitmirror git clone
    git@source-repository.com:example/example.git
    ```

3. 创建镜像仓库：

    ```shell
    sudo -u gitmirror mkdir example-mirror && cd example-mirror
    sudo -u gitmirror git clone --mirror git@source-repository.com:example/example.git
    cd example.git/
    sudo -u gitmirror git remote add --mirror=push target git@target-repository.com:example/example-mirror.git
    ````

4. 创建同步脚本，测试：

    ```shell
    cat <<EOF > /tmp/mirror-example.sh
    #!/bin/bash
    cd /home/gitmirror/example-mirror/example.git/
    git fetch --prune origin
    git push --mirror target
    EOF
    chmod 755 /tmp/mirror-example.sh
    sudo -u gitmirror cp -a /tmp/mirror-example.sh /home/gitmirror/example-mirror/mirror-example.sh

    sudo -u gitmirror /home/gitmirror/example-mirror/mirror-example.sh
    ```

5. 添加同步脚本到 `/etc/crontab` (定时任务，例如 每隔30分钟运行)：

    ```
    */30 * * * *  gitmirror  /home/gitmirror/example-mirror/mirror-example.sh
    ```

## GitLab

GitLab 仓库和源仓库进行双向同步需要不同的配置

由于 GitLab 使用钩子来追踪仓库中的修改，所以需要一个单独的卫星(satellite)仓库用来拉取源仓库的修改并推送到 GitLab 来进行双向同步，反之亦然

以下操作假设已经安装了 GitLab Omnibus

### 配置用于同步的卫星仓库

1. 切换到 GitLab 的 `git` 用户，生成SSH密钥(key)：

    ```shell
    sudo su git
    ssh-keygen
    ```

2. 在 GitLab 页面上创建一个专门用于镜像的账户，在账户配置中上传SSH公钥

3. 在 GitLab 页面上创建镜像项目，给镜像账户 _Master_ 权限

4. 复制SSH公钥到源仓库的SSH已验证公钥(authorized keys)里：

    ```shell
    scp ~/.ssh/id_rsa.pub user@origin-repository-host:
    ssh user@origin-repository-host
    cat id_rsa.pub >> ~/.ssh/authorized_keys
    logout
    ```

5. _(可选)_ 如果源仓库的用户名和 `git` 不同，配置一个SSH host别名：

    ```shell
    cat >> ~/.ssh/config << EOT
    Host gitmirror-origin-host
    User user
    HostName origin-repository-host
    EOT
    ```

6. 在 GitLab 服务器中确保SSH服务器接受来自`localhost`的连接

7. 为 `git` 用户配置同步工具工作空间，并配置 `git-mirror`：

    ```shell
    sudo mkdir /var/opt/gitlab/mirroring-tools
    sudo chown git: /var/opt/gitlab/mirroring-tools
    sudo su git
    cd ~/mirroring-tools
    mkdir utils
    
    cd utils
    git clone https://github.com/mrts/git-mirror.git
    cd git-mirror/scripts/satellite
    
    # change the substituted values below according to your needs
    sed -i 's#CONF_ORIGIN_URL=.*#CONF_ORIGIN_URL=origin-repository-host:git/repo.git#' \
        synchronize-git-repositories-with-satellite.config
    sed -i 's#CONF_OTHER_URL=.*#CONF_OTHER_URL=localhost:mirror/repo.git#' \
        synchronize-git-repositories-with-satellite.config
    sed -i 's#CONF_GITDIR=.*#CONF_GITDIR=/var/opt/gitlab/mirroring-tools/repo.git#' \
        synchronize-git-repositories-with-satellite.config
    sed -i 's#CONF_OTHER_GITDIR=.*#CONF_OTHER_GITDIR=/var/opt/gitlab/git-data/repositories/mirror/repo.git#' \
        synchronize-git-repositories-with-satellite.config
    ```

8. 运行配置脚本：

    ```shell
    ./setup-synchronize-git-repositories-with-satellite.sh
    ```

    * 配置脚本会进行以下操作：

        1. 在卫星仓库中配置源仓库和 GitLab 仓库
        2. 在 GitLab 仓库中配置 `post-receive` 钩子来推送更改
        3. 打印应添加到 `crontab` 的配置，用于运行定时同步

    * 确保同步工作不用密码，即在配置过程中不应该需要密码

9. 将配置脚本运行最后打印出的一行添加到 `crontab`：

    ```shell
    sudo sh -o noglob -c 'echo "*/1 * * * *  git  ...path..." >> /etc/crontab'
    ```

10. 测试并检查日志

    1. 验证 GitLab 上所有的分支都存在并包含正确的提交记录
    2. 创建一个合并请求并将它合并到 GitLab，验证合并会被立即同步到源仓库
    3. 在源仓库中提交修改到任意一个分支，确保修改会被同步到 GitLab 
    after cron has run.
    4. 重写 `master` 分支的提交历史，并在 GitLab 中删除 `master` 分支，验证这**不会**被同步到源仓库，同时会输出错误信息到日志
    5. 在 GitLab 中删除和创建任意一个分支，验证删除和创建会被立即同步到源仓库
    6. 在源仓库中删除和创建分支，验证这些修改会在 `crontab` 运行定时任务的被同步到 GitLab

11. 搞定 :)!

### 注意

在 GitLab页面上创建或删除分支似乎不会触发 `post-receive` 钩子，所以已经删除的分支将会在源仓库下次同步时重新出现，这个问题已经作为[bug #1156](https://gitlab.com/gitlab-org/gitlab-ce/issues/1156)提交到 GitLab 的问题追踪(issue tracker)

## 协议

[MIT](LICENSE)

这个项目包含以下第三方开源组件：
```
─── scripts
    ├── git-update-hook-for-protecting-branches-and-tags
    ├── satellite
    │   ├── git-post-receive-hook-for-updating-satellite
    │   ├── setup-synchronize-git-repositories-with-satellite.sh
    │   ├── synchronize-git-repositories-with-satellite.config
    │   └── synchronize-git-repositories-with-satellite.sh
    ├── simple
    │   ├── setup-synchronize-git-mirror.sh
    │   ├── synchronize-git-mirror.config
    │   └── synchronize-git-mirror.sh
    └── utils-lib.sh
```
每一个组件都有自己的协议，请查看 [licenses/LICENSE](licenses/LICENSE).