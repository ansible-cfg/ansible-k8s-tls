-------------------------------------------------------------------------------
## 部署前 ##
1. 己安装好centos 7.3/7.4操作系统
2. 准备ansible环境  
说明: ansible和离线源需要一台额外的主机, 安装完成后即可回收主机

``` shell
# 安装pip
yum -y install python-setuptools
easy_install pip
# 安装ansible
pip install ansible
```

3. 下载ansible-k8s-tls ansible playbook
``` shell
git clone https://github.com/liujun1990/ansible-k8s-tls
cd ansible-k8s-tls
```
> 以下操作都以ansible-k8s-tls为basedir

4. 配置认证信息
	- i. 复制以下内容生成vault.sh脚本
	``` shell
	cat <<'EOF' > vault.sh
	VAULT_ID='myVAULT@2018'
	echo $VAULT_ID > ~/.vault_pass.txt

	ANSIBLE_USER='root' # ssh用户名
	ANSIBLE_PASSWORD='root' # ssh用户密码

	ansible-vault encrypt_string --vault-id ~/.vault_pass.txt $ANSIBLE_USER --name 'vault_ansible_user' | tee dev/group_vars/vault
	ansible-vault encrypt_string --vault-id ~/.vault_pass.txt $ANSIBLE_PASSWORD --name 'vault_ansible_password' | tee -a dev/group_vars/vault
	EOF
	```
	**注意:** 请务必修改脚本中的ssh用户名密码及dce认证用户名密码与实际环境匹配
	- ii. 执行脚本
	``` shell
	bash vault.sh
	```
  


-------------------------------------------------------------------------------
## 安装k8s ##
### 1. 创建TLS证书和秘钥 ###
> cfssl下载地址
> etcd,kube-apiserver的ip地址替换成与实际环境匹配, 如etcd有3台(192.168.130.11-13), 同时192.168.130.11作为kube-apiserver
``` shell
...
function kubernetes_gen {
	cat > kubernetes-csr.json <<-EOF
	{
	    "CN": "kubernetes",
	    "hosts": [
	      "192.168.130.11",
	      "192.168.130.12",
	      "192.168.130.13",
	      "127.0.0.1",
	      "10.254.0.1",
	      "kubernetes",
	      "kubernetes.default",
	      "kubernetes.default.svc",
	      "kubernetes.default.svc.cluster",
	      "kubernetes.default.svc.cluster.local"
	    ],
...
```
``` shell
cd tools
bash tls_gen.sh
```
### 2. 创建kubeconfig文件 ###
``` shell
cd tools
bash kubeconfig_gen.sh
```
### 3. 创建base64格式的etcd证书变量文件 ###
``` shell
cd tools
bash etcd_tls2base64.sh
```
### 4. 定义变量 ###
**dev/group_vars/all**    
> 镜像仓库地址，kube-apiserver等二进制文件下载地址请根据实际环境填写
**dev/hosts**  
> etcd是etcd集群节点组
> kubernetes-master是kubenetes master节点组
> kubernetes-node是kubenetes node节点组
> kubernetes-client是kubectl命令执行的节点，通常为kubernetes master节点, 主要用来执行kubectl命令
``` ini
[etcd]
192.168.130.11
192.168.130.12
192.168.130.13

[kubernetes-master]
192.168.130.11

[kubernetes-node]
192.168.130.11
192.168.130.12
192.168.130.13

[kubernetes-client]
192.168.130.11
```
### 4. 一键安装k8s ###
``` shell
ansible-playbook -i dev/hosts --vault-password-file ~/.vault_pass.txt --extra-vars install_or_uninstall=install one_step_install.yml
```
  


-------------------------------------------------------------------------------
## 卸载k8s ##
``` shell
ansible-playbook -i dev/hosts --vault-password-file ~/.vault_pass.txt --extra-vars install_or_uninstall=uninstall one_step_uninstall.yml
```
