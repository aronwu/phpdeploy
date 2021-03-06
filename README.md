# phpdeploy

**phpdeploy**使用**ansible**快速发布PHP项目，自动从git仓库检出代码，自动检查composer.json是否有更新自动更新vendor, 同时区分多个开发，测试，产线环境以及一键快速回滚代码到上个版本。

## 全局配置参数

文件group_vars/all
```
---
# deploy global vars
ans_source_path: "/Users/wuxinjun/git/test/phpdeploy"
ans_version_dir: "release"
ans_current_dir: "current"
ans_current_via: "symlink"
ans_shared_paths: []
ans_shared_files: []
ans_keep_releases: 10
ans_deploy_via: "rsync"
ans_remove_rolled_back: true
ans_rsync_extra_params: " --exclude-from=.excludes "
ans_composer_path: "/usr/bin/composer"
# package manage 
ans_pkg_mgr: yum
ans_remote_user: deploy

```
1. ans_source_path: 整个项目所有从git库检出的项目代码source目录
2. ans_version_dir: 发布到远程主机的项目release目录名称
3. ans_current_dir: 发布到远程主机的项目current目录名称
4. ans_current_via: current目录链接到最新release版本方法
5. ans_keep_releases： release目录保留最多历史版本数目
6. ans_deploy_via： 远程同步脚本方法
7. ans_remove_rolled_back： 当回滚代码时是否删除当前版本的代码
8. ans_composer_path：本机composer命令路径
9. ans_pkg_mgr: 包管理命令
10. ans_remote_user: 远程执行用户



## 主机组发布参数

在group_vars目录下，默认有个apihosts主机组配置参数

```
---
# deploy vars
ans_deploy_from: "{{ ans_source_path }}/source/api/"
ans_deploy_to: "/opt/demo/api"
ans_git_repo: git@bitbucket.org:xxx/xxx.git
ans_git_branch: "{{ git_branch | default('master')}}"
ans_rsync_set_remote_user: deploy
ans_rsync_extra_params:
ans_env_file: "{{ ans_source_path }}/envs/{{ env }}/config/api/.env"

# storage path
storage_dir: "storage"

# nginx configuration
nginx_domain_name: "api.demo.com"
nginx_log_root: "/var/log/nginx"
nginx_document_root: "{{ ans_deploy_to }}/current/public"

```

1. ans_deploy_from: 从git库检出的代码本地目录
2. ans_deploy_to: 代码发布到目标服务器的目录
3. ans_git_repo: git库地址
4. ans_git_branch: git库检出分支，默认是master分支
5. ans_rsync_set_remote_user: 远程同步代码的账号
6. ans_rsync_extra_params: 远程同步脚本额外参数
7. ans_env_file: 不同环境参数配置文件
8. storage_dir: 发布后所有版本共同缓存目录
9. nginx_domain_name: 网站域名
10. nginx_log_root: 网站日志目录
11. nginx_document_root: 网站根目录，需要指向发布目录current下或者子目录



## 发布目录结构
> **ans_deploy_to** 目录下包含   
>   -current -> ./release/20160729035641Z  
>   -release  
>   &nbsp;&nbsp;&nbsp;&nbsp;- 20160726100729Z   
>   &nbsp;&nbsp;&nbsp;&nbsp;- 20160728091239Z  
>   &nbsp;&nbsp;&nbsp;&nbsp;- 20160729035641Z  
>   -shared  
>   -storage 

1. current目录指向release下当前最新的版本。  
2. release目录包含当前发布所有版本历史。
3. storage目录包含项目所有存储包括缓存等。

### nginx配置

```
server {
    listen       80;
    server_name  {{ nginx_domain_name }};
    charset utf-8;
    access_log  {{ nginx_log_root }}/{{ nginx_domain_name }}.access.log  main;
    error_log   {{ nginx_log_root }}/{{ nginx_domain_name }}.error.log;
    root {{ nginx_document_root }};
    location / {
	  index index.php index.html index.htm;
    }
    location ~ \.php$ {
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $realpath_root$fastcgi_script_name;
        fastcgi_param  WWW_ROOT $realpath_root;
        include        fastcgi_params;
    }
}


```


1. 其中SCRIPT_FILENAME  $realpath_root$fastcgi_script_name配置用的是realpath_root
2. nginx document root 指向发布目录的current下。

##  不同环境

phpdeploy可以支持开发，测试，产线多个环境发布任务。envs目录默认包含了开发，测试和产线配置，每个目录下有hosts文件可以配置不同环境下的主机。每个环境下包含不同ansible组的环境配置文件.env.

## 执行发布命令

```
ansible-playbook -i envs/dev/hosts apideploy.yml --extra-vars "deploy_env=dev git_branch=master"

```


