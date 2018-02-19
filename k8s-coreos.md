## kubernetes-vagrant-coreos-cluster
Kubernetes cluster (for testing purposes) made easy with Vagrant and CoreOS.
参考 [pires/kubernetes-vagrant-coreos-cluster](https://github.com/pires/kubernetes-vagrant-coreos-cluster).

[Deploy Kubernetes: The Ultimate Guide](https://platform9.com/docs/deploy-kubernetes-the-ultimate-guide/)

---
### 修改说明
###### Update Log
Update 1: 为网路环境修改
* 新增说明文档： `k8s-coreos.md`、`k8s-coreos.html`
* 修改 `Vagrantfile`, 修改 `MASTER_CPUS` / `BASE_IP_ADDR` 设置 与 windows 重定向问题.
  不使用 `plugins/dns/coredns/deploy.sh`, 解决 windows 下脚本重定向输出问题
* `setup.tmpl` 文件, 修改 “`KUB_URL=`” 为本地下载地址  
* 以下文件中 “`gcr.io/google_containers/hyperkube-amd64`” 修改为 “`wangwg2/hyperkube-amd64`”
  `master.yaml`
  `manifests/master-apiserver-rbac.yaml`
  `manifests/master-apiserver.yaml`
  `manifests/master-controller-manager.yaml`
  `manifests/master-proxy.yaml`
  `manifests/master-scheduler.yaml`
  `manifests/node-proxy.yaml`
* 修改 `master.yaml`, 解决 `gcr.io/google_containers/pause-amd64:3.0` 被墙
  为 `\hyperkube kubelet` 增加 `--pod-infra-container-image=wangwg2/pause-amd64:3.0 \`
* `docker logs -f container_name` 查看容器日志分析问题。

###### Vagrant 虚机设置
`Vagrantfile` 文件
```ruby
# 解决CPU不足
MASTER_CPUS = ENV['MASTER_CPUS'] || 1
NODE_CPUS = ENV['NODE_CPUS'] || 1
# IP网络段
BASE_IP_ADDR = ENV['BASE_IP_ADDR'] || "192.168.99"
```

###### Window sh 重定向问题 
修改 `Vagrantfile`, 不使用 `plugins/dns/coredns/deploy.sh`
`Vagrantfile`
```ruby
  if DNS_PROVIDER == "kube-dns"
    # create dns-deployment.yaml file
    dnsFile = "#{__dir__}/temp/dns-deployment.yaml"
    # .....
  else if DNS_PROVIDER == "coredns"
      # --- 修改前 ---
      system "#{__dir__}/plugins/dns/coredns/deploy.sh 10.100.0.10/24 #{DNS_DOMAIN} #{__dir__}/plugins/dns/coredns/coredns.yaml.sed > #{__dir__}/temp/coredns-deployment.yaml"
      # --- 修改后 ---
      system "cp #{__dir__}/plugins/dns/coredns/coredns.yaml.sed #{__dir__}/temp/coredns-deployment.yaml"
      system "sed -i -e s/CLUSTER_DNS_IP/10.100.0.10/g -e s/CLUSTER_DOMAIN/#{DNS_DOMAIN}/g -e s?SERVICE_CIDR?10.100.0.10\/24?g #{__dir__}/temp/coredns-deployment.yaml"
    end
  end
```

###### kubectl 下载慢 
`setup.tmpl` 文件
解决 `bukectl`下载慢问题。预先下载到指定目录。
`https://storage.googleapis.com/kubernetes-release/release/v${KUBERNETES_VERSION}/bin/linux/amd64/kubectl`

启动下载服务器：(192.168.99.23)
`docker run --name archives-nginx -v /ansible/archives:/usr/share/nginx/html:ro -p 8080:80 -d nginx`

修改 `setup.tmpl`
`KUB_URL="http://192.168.99.23/kubectl"`

###### Docker Registry
`docker run -d -v /data/docker/voldata/registry:/data -p 5000:5000 --restart=always --name registry registry:2`

`gcr.io/google_containers/hyperkube-amd64:__RELEASE__`
`wangwg2/hyperkube-amd64:__RELEASE__`

###### DOCKER_OPTIONS
`Vagrantfile` 文件
```ruby
DOCKER_OPTIONS = ENV['DOCKER_OPTIONS'] || '--registry-mirror=https://4ue5z1dy.mirror.aliyuncs.com'
```

###### master.yml
hyperkube kubelet 会 pull 镜像 `gcr.io/google_containers/pause-amd64:3.0`。
解决镜像被墙问题，改为 `wangwg2/pause-amd64:3.0`。
增加 `--pod-infra-container-image=wangwg2/pause-amd64:3.0 \`
```yaml
ExecStart=/usr/bin/docker run \
  --volume=/:/rootfs:ro \
  --volume=/sys:/sys:ro \
  --volume=/etc/kubernetes:/etc/kubernetes:ro \
  --volume=/var/lib/docker/:/var/lib/docker:rw \
  --volume=/var/lib/kubelet/:/var/lib/kubelet:rw \
  --volume=/var/run:/var/run:rw \
  --net=host \
  --pid=host \
  --privileged=true \
  --name=kubelet \
  -d \
  wangwg2/hyperkube-amd64:__RELEASE__ \
    /hyperkube kubelet \
    --address=$private_ipv4 \
    --network-plugin= \
    --pod-infra-container-image=wangwg2/pause-amd64:3.0 \
    --register-schedulable=false \
    --allow-privileged=true \
    --pod-manifest-path=/etc/kubernetes/manifests \
    --hostname-override=$public_ipv4 \
    --cluster_dns=10.100.0.10 \
    --cluster_domain=__DNS_DOMAIN__ \
    --kubeconfig=/etc/kubernetes/master-kubeconfig.yaml
```
docker run 说明：
* `--net` 指定容器的网络类型：`bridge` / `host` / `none` / `container`
* `--pid` PID 命名空间
* `--privileged` 给容器扩展特权

kubelet 说明：
* `--address`
  kubelet 服务监听的地址 (默认: 0.0.0.0)
* `--pod-infra-container-image`
  基础镜像地址，每个 pod 最先启动的容器，会配置共享的网络
  (默认: gcr.io/google_containers/pause-amd64:3.0)
* `--register-schedulable=false`
* `--allow-privileged`
  是否允许容器运行在 privileged 模式 (默认:false)
* `--pod-manifest-path`
  本地 manifest 文件的路径或者目录 (默认:"")
* `--hostname-override`
  指定 hostname，如果非空会使用这个值作为节点在集群中的标识
* `--cluster-dns stringSlice`
  逗号分隔的 DNS 服务器 IP 地址. This value is used for containers DNS server in case of Pods with "dnsPolicy=ClusterFirst". Note: all DNS servers appearing in the list MUST serve the same set of records otherwise name resolution within the cluster may not work correctly. There is no guarantee as to which DNS server may be contacted for name resolution.
* `--cluster-domain string`
  Domain for this cluster. If set, kubelet will configure all containers to search this domain in addition to the host's search domains  
* `--kubeconfig string`
  kubeconfig 文件, 说明如何连接 API server. (默认："/var/lib/kubelet/kubeconfig")



---
### Vagrantfile 解析
###### Vagrantfile (1)
预备 （模块，插件， 定义变量）
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'fileutils'
require 'net/http'
require 'open-uri'
require 'json'
require 'date'
require 'pathname'

class Module
  def redefine_const(name, value)
    __send__(:remove_const, name) if const_defined?(name)
    const_set(name, value)
  end
end

module OS
  def OS.windows?
    (/cygwin|mswin|mingw|bccwin|wince|emx/ =~ RUBY_PLATFORM) != nil
  end

  def OS.mac?
   (/darwin/ =~ RUBY_PLATFORM) != nil
  end

  def OS.unix?
    !OS.windows?
  end

  def OS.linux?
    OS.unix? and not OS.mac?
  end
end

required_plugins = %w(vagrant-triggers)

# check either 'http_proxy' or 'HTTP_PROXY' environment variable
enable_proxy = !(ENV['HTTP_PROXY'] || ENV['http_proxy'] || '').empty?
if enable_proxy
  required_plugins.push('vagrant-proxyconf')
end

if OS.windows?
  required_plugins.push('vagrant-winnfsd')
end

required_plugins.push('vagrant-timezone')

required_plugins.each do |plugin|
  need_restart = false
  unless Vagrant.has_plugin? plugin
    system "vagrant plugin install #{plugin}"
    need_restart = true
  end
  exec "vagrant #{ARGV.join(' ')}" if need_restart
end

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"
Vagrant.require_version ">= 1.8.6"

MASTER_YAML = File.join(File.dirname(__FILE__), "master.yaml")
NODE_YAML = File.join(File.dirname(__FILE__), "node.yaml")

# AUTHORIZATION MODE is a setting for enabling or disabling RBAC for your Kubernetes Cluster
# The default mode is ABAC.
AUTHORIZATION_MODE = ENV['AUTHORIZATION_MODE'] || 'AlwaysAllow'

if AUTHORIZATION_MODE == "RBAC"
  CERTS_MASTER_SCRIPT = File.join(File.dirname(__FILE__), "tls/make-certs-master-rbac.sh")
else
  CERTS_MASTER_SCRIPT = File.join(File.dirname(__FILE__), "tls/make-certs-master.sh")
end

CERTS_MASTER_CONF = File.join(File.dirname(__FILE__), "tls/openssl-master.cnf.tmpl")
CERTS_NODE_SCRIPT = File.join(File.dirname(__FILE__), "tls/make-certs-node.sh")
CERTS_NODE_CONF = File.join(File.dirname(__FILE__), "tls/openssl-node.cnf.tmpl")

MANIFESTS_DIR = Pathname.getwd().join("manifests")

USE_DOCKERCFG = ENV['USE_DOCKERCFG'] || false
DOCKERCFG = File.expand_path(ENV['DOCKERCFG'] || "~/.dockercfg")

# DOCKER_OPTIONS = ENV['DOCKER_OPTIONS'] || ''
DOCKER_OPTIONS = ENV['DOCKER_OPTIONS'] || '--registry-mirror=https://4ue5z1dy.mirror.aliyuncs.com'


KUBERNETES_VERSION = ENV['KUBERNETES_VERSION'] || '1.9.2'

CHANNEL = ENV['CHANNEL'] || 'alpha'

#if CHANNEL != 'alpha'
#  puts "============================================================================="
#  puts "As this is a fastly evolving technology CoreOS' alpha channel is the only one"
#  puts "expected to behave reliably. While one can invoke the beta or stable channels"
#  puts "please be aware that your mileage may vary a whole lot."
#  puts "So, before submitting a bug, in this project, or upstreams (either kubernetes"
#  puts "or CoreOS) please make sure it (also) happens in the (default) alpha channel."
#  puts "============================================================================="
#end

COREOS_VERSION = ENV['COREOS_VERSION'] || 'latest'
upstream = "http://#{CHANNEL}.release.core-os.net/amd64-usr/#{COREOS_VERSION}"
if COREOS_VERSION == "latest"
  upstream = "http://#{CHANNEL}.release.core-os.net/amd64-usr/current"
  url = "#{upstream}/version.txt"
  Object.redefine_const(:COREOS_VERSION,
    open(url).read().scan(/COREOS_VERSION=.*/)[0].gsub('COREOS_VERSION=', ''))
end

NODES = ENV['NODES'] || 2

MASTER_MEM = ENV['MASTER_MEM'] || 1024
MASTER_CPUS = ENV['MASTER_CPUS'] || 1

NODE_MEM= ENV['NODE_MEM'] || 2048
NODE_CPUS = ENV['NODE_CPUS'] || 1

# BASE_IP_ADDR = ENV['BASE_IP_ADDR'] || "172.17.8"
BASE_IP_ADDR = ENV['BASE_IP_ADDR'] || "192.168.99"

DNS_PROVIDER = ENV['DNS_PROVIDER'] || "coredns"
DNS_DOMAIN = ENV['DNS_DOMAIN'] || "cluster.local"

SERIAL_LOGGING = (ENV['SERIAL_LOGGING'].to_s.downcase == 'true')
GUI = (ENV['GUI'].to_s.downcase == 'true')
USE_KUBE_UI = ENV['USE_KUBE_UI'] || false

BOX_TIMEOUT_COUNT = ENV['BOX_TIMEOUT_COUNT'] || 50

if enable_proxy
  HTTP_PROXY = ENV['HTTP_PROXY'] || ENV['http_proxy']
  HTTPS_PROXY = ENV['HTTPS_PROXY'] || ENV['https_proxy']
  NO_PROXY = ENV['NO_PROXY'] || ENV['no_proxy'] || "localhost"
end

REMOVE_VAGRANTFILE_USER_DATA_BEFORE_HALT = (ENV['REMOVE_VAGRANTFILE_USER_DATA_BEFORE_HALT'].to_s.downcase == 'true')
# if this is set true, remember to use --provision when executing vagrant up / reload

# Read YAML file with mountpoint details
MOUNT_POINTS = YAML::load_file(File.join(File.dirname(__FILE__), "synced_folders.yaml"))
```

说明： ruby模块，插件， 定义变量
```ruby
## ruby 模块： fileutils net/http ...
## 插件：vagrant-triggers vagrant-proxyconf vagrant-winnfsd vagrant-timezone
# def OS.windows
# MASTER_YAML         = master.yaml
# NODE_YAML           = node.yaml
# CERTS_MASTER_SCRIPT = tls/make-certs-master.sh 或 tls/make-certs-master.sh
# CERTS_MASTER_CONF   = tls/openssl-master.cnf.tmpl
# CERTS_NODE_SCRIPT   = tls/make-certs-node.sh
# CERTS_NODE_CONF     = tls/openssl-node.cnf.tmpl
# DOCKERCFG           = ~/.dockercfg
# DOCKER_OPTIONS      = 
# KUBERNETES_VERSION  = 1.9.2
# CHANNEL             = alpha
# COREOS_VERSION      = latest
# NODES               = 2
# MASTER_MEM          = 1024
# MASTER_CPUS         = 1
# NODE_MEM            = 2048
# NODE_CPUS           = 1
# BASE_IP_ADDR        = 192.168.99
# DNS_PROVIDER        = coredns
# DNS_DOMAIN          = cluster.local
# SERIAL_LOGGING      = false
# GUI                 = false
# USE_KUBE_UI         = false
# BOX_TIMEOUT_COUNT   = 50
# HTTP_PROXY          = ENV['HTTP_PROXY']
# HTTPS_PROXY         = ENV['HTTPS_PROXY']
# NO_PROXY            = "localhost"
# REMOVE_VAGRANTFILE_USER_DATA_BEFORE_HALT = false
# MOUNT_POINTS        = synced_folders.yaml
```

###### Vagrantfile (2)
Vagrant.configure
```ruby
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # always use host timezone in VMs
  config.timezone.value = :host

  # always use Vagrants' insecure key
  config.ssh.insert_key = false
  config.ssh.forward_agent = true

  config.vm.box = "coreos-#{CHANNEL}"
  config.vm.box_version = "= #{COREOS_VERSION}"
  config.vm.box_url = "#{upstream}/coreos_production_vagrant.json"

  ["vmware_fusion", "vmware_workstation"].each do |vmware|
    config.vm.provider vmware do |v, override|
      override.vm.box_url = "#{upstream}/coreos_production_vagrant_vmware_fusion.json"
    end
  end

  config.vm.provider :parallels do |vb, override|
    override.vm.box = "AntonioMeireles/coreos-#{CHANNEL}"
    override.vm.box_url = "https://vagrantcloud.com/AntonioMeireles/coreos-#{CHANNEL}"
  end

  config.vm.provider :virtualbox do |v|
    # On VirtualBox, we don't have guest additions or a functional vboxsf
    # in CoreOS, so tell Vagrant that so it can be smarter.
    v.check_guest_additions = false
    v.functional_vboxsf     = false
  end
  config.vm.provider :parallels do |p|
    p.update_guest_tools = false
    p.check_guest_tools = false
  end

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  # setup VM proxy to system proxy environment
  if Vagrant.has_plugin?("vagrant-proxyconf") && enable_proxy
    config.proxy.http = HTTP_PROXY
    config.proxy.https = HTTPS_PROXY
    # most http tools, like wget and curl do not undestand IP range
    # thus adding each node one by one to no_proxy
    no_proxies = NO_PROXY.split(",")
    (1..(NODES.to_i + 1)).each do |i|
      vm_ip_addr = "#{BASE_IP_ADDR}.#{i+100}"
      Object.redefine_const(:NO_PROXY,
        "#{NO_PROXY},#{vm_ip_addr}") unless no_proxies.include?(vm_ip_addr)
    end
    config.proxy.no_proxy = NO_PROXY
    # proxyconf plugin use wrong approach to set Docker proxy for CoreOS
    # force proxyconf to skip Docker proxy setup
    config.proxy.enabled = { docker: false }
  end

  (1..(NODES.to_i + 1)).each do |i|
    if i == 1
      hostname = "master"
      ETCD_SEED_CLUSTER = "#{hostname}=http://#{BASE_IP_ADDR}.#{i+100}:2380"
      cfg = MASTER_YAML
      memory = MASTER_MEM
      cpus = MASTER_CPUS
      MASTER_IP="#{BASE_IP_ADDR}.#{i+100}"
    else
      hostname = "node-%02d" % (i - 1)
      cfg = NODE_YAML
      memory = NODE_MEM
      cpus = NODE_CPUS
    end

    config.vm.define vmName = hostname do |kHost|
      # ######################################################################
      # ## 核心部分
      # ######################################################################
    end
  end
end  
```

###### Vagrantfile (3)
Vagrant.configure - config.vm.define
```ruby
config.vm.define vmName = hostname do |kHost|
  kHost.vm.hostname = vmName

  # suspend / resume is hard to be properly supported because we have no way
  # to assure the fully deterministic behavior of whatever is inside the VMs
  # when faced with XXL clock gaps... so we just disable this functionality.
  kHost.trigger.reject [:suspend, :resume] do
    info "'vagrant suspend' and 'vagrant resume' are disabled."
    info "- please do use 'vagrant halt' and 'vagrant up' instead."
  end

  config.trigger.instead_of :reload do
    exec "vagrant halt && vagrant up"
    exit
  end

  # vagrant-triggers has no concept of global triggers so to avoid having
  # then to run as many times as the total number of VMs we only call them
  # in the master (re: emyl/vagrant-triggers#13)...
  # ## master 虚拟机
  if vmName == "master"
    # ## trigger.before [:up, :provision]
    kHost.trigger.before [:up, :provision] do
      info "#{Time.now}: setting up Kubernetes master..."
      info "Setting Kubernetes version #{KUBERNETES_VERSION}"

      # create setup file
      setupFile = "#{__dir__}/temp/setup"
      # ## create temp/setup 

      if DNS_PROVIDER == "kube-dns"
        # ## create dns-deployment.yaml file
        # ## 根据 /plugins/dns/kube-dns/dns-deployment-rbac.yaml.tmpl
        # ##   或 /plugins/dns/kube-dns/dns-deployment.yaml.tmpl
      else if DNS_PROVIDER == "coredns"
        # ## 拷贝 plugins/dns/coredns/coredns.yaml.sed 到 temp/coredns-deployment.yaml
        # ## 编辑 temp/coredns-deployment.yaml, 替换变量值
      end
    end

    # ## trigger.after [:up, :resume]
    kHost.trigger.after [:up, :resume] do
      unless OS.windows?
        # ## 运行 ssh-add ~/.vagrant.d/insecure_private_key
        # ## 运行 rm -rf ~/.fleetctl/known_hosts
      end
    end

    # ## trigger.after [:up]
    kHost.trigger.after [:up] do
      info "Waiting for Kubernetes master to become ready..."
      # ## 等待 Kubernetes master 就绪

      info "Installing kubectl for the Kubernetes version we just bootstrapped..."
      if OS.windows?
        # ## 运行 sudo -u core /bin/sh /home/core/kubectlsetup install
      else
        # ## 运行 ./temp/setup install
      end

      # set cluster
      if OS.windows?
        # ## 运行
        # ## /opt/bin/kubectl config set-cluster default-cluster --server=https://#{MASTER_IP} --certificate-authority=/vagrant/artifacts/tls/ca.pem
        # ## /opt/bin/kubectl config set-credentials default-admin --certificate-authority=/vagrant/artifacts/tls/ca.pem --client-key=/vagrant/artifacts/tls/admin-key.pem --client-certificate=/vagrant/artifacts/tls/admin.pem
        # ## /opt/bin/kubectl config set-context local --cluster=default-cluster --user=default-admin
        # ## /opt/bin/kubectl config use-context local
      else
        # ## 运行
        # ## kubectl config set-cluster default-cluster --server=https://#{MASTER_IP} --certificate-authority=artifacts/tls/ca.pem
        # ## kubectl config set-credentials default-admin --certificate-authority=artifacts/tls/ca.pem --client-key=artifacts/tls/admin-key.pem --client-certificate=artifacts/tls/admin.pem
        # ## kubectl config set-context local --cluster=default-cluster --user=default-admin
        # ## kubectl config use-context local
      end

      info "Configuring Kubernetes DNS..."
      # ## 配置 Kubernetes DNS
      if DNS_PROVIDER == "kube-dns"
        # ## 运行 
        # ## /opt/bin/kubectl create -f /home/core/dns-deployment.yaml
        # ## /opt/bin/kubectl create -f /home/core/dns-configmap.yaml
        # ## /opt/bin/kubectl create -f /home/core/dns-service.yaml
      else if DNS_PROVIDER == "coredns"
        # ## 运行 
        # ## /opt/bin/kubectl create -f /home/core/coredns-deployment.yaml
      end

      if USE_KUBE_UI
        info "Configuring Kubernetes dashboard..."
        # ## 配置 Kubernetes dashboard
        # ## 运行 
        # ## /opt/bin/kubectl create -f /home/core/dashboard-deployment.yaml
        # ## /opt/bin/kubectl create -f /home/core/dashboard-service.yaml
        info "Kubernetes dashboard will be available at http://#{MASTER_IP}:8080/ui"
      end

    end

    # copy setup files to master vm if host is windows
    if OS.windows?
      # ## 拷贝 temp/setup 到 /home/core/kubectlsetup

      if DNS_PROVIDER == "kube-dns"
        # ## 拷贝 plugins/dns/kube-dns/dns-configmap.yaml 到 /home/core/dns-configmap.yaml
        # ## 拷贝 temp/dns-deployment.yaml 到 /home/core/dns-deployment.yaml
        # ## 拷贝 plugins/dns/kube-dns/dns-service.yaml 到 /home/core/dns-service.yaml
        # ## 拷贝 plugins/dns/kube-dns/dns-service-rbac.yaml 到 /home/core/dns-service-rbac.yaml
      else if DNS_PROVIDER == "coredns"
        # ## 拷贝 temp/coredns-deployment.yaml 到 /home/core/coredns-deployment.yaml
      end

      if USE_KUBE_UI
        # ## 拷贝 plugins/dashboard/dashboard-deployment.yaml 到 /home/core/dashboard-deployment.yaml
        # ## 拷贝 plugins/dashboard/dashboard-service.yaml 到 /home/core/dashboard-service.yaml
      end
    end

    # clean temp directory after master is destroyed
    # ## trigger.after [:destroy]
    kHost.trigger.after [:destroy] do
      # ## 清理 宿主机 temp/* 和 artifacts/tls/*
    end
  end

  # ## node 虚拟机
  if vmName == "node-%02d" % (i - 1)
    # ## trigger.before [:up, :provision]
    kHost.trigger.before [:up, :provision] do
      info "#{Time.now}: setting up node..."
    end

    # ## trigger.after [:up]
    kHost.trigger.after [:up] do
      info "Waiting for Kubernetes minion [node-%02d" % (i - 1) + "] to become ready..."
      # ## 等待 node 虚拟机准备好
      if hasResponse
        info "#{Time.now}: successfully deployed #{vmName}"
      else
        info "#{Time.now}: failed to deploy #{vmName} within timeout count of #{BOX_TIMEOUT_COUNT}"
      end
    end
  end

  kHost.trigger.before [:halt, :reload] do
    # ## 清除 /var/lib/coreos-vagrant/vagrantfile-user-data
  end

  if SERIAL_LOGGING
    # ## 虚拟机 serial log 设置
  end

  ["vmware_fusion", "vmware_workstation", "virtualbox"].each do |h|
    # ## set gui = GUI
  end
  ["vmware_fusion", "vmware_workstation"].each do |h|
    # ## set vm vmx["memsize"], vmx["numvcpus"], vmx['virtualHW.version']
  end
  ["parallels", "virtualbox"].each do |h|
    # ## set vm memory, cpus
  end

  kHost.vm.network :private_network, ip: "#{BASE_IP_ADDR}.#{i+100}"

  # you can override this in synced_folders.yaml
  kHost.vm.synced_folder ".", "/vagrant", disabled: true

  begin
    MOUNT_POINTS.each do |mount|
    # ## mount 文件系统
    end
  rescue
  end

  if USE_DOCKERCFG && File.exist?(DOCKERCFG)
    # ## 拷贝 DOCKERCFG 到 /home/core/.dockercfg
    # ## 拷贝 /home/core/.dockercfg 到 /root/.dockercfg
  end

  # Copy TLS stuff
  # ## [provision] 处理 TLS 
  if vmName == "master"
    # 拷贝 tls/make-certs-master-rbac.sh 到 /tmp/make-certs.sh
    # 拷贝 tls/openssl-master.cnf.tmpl 到 /tmp/openssl.cnf
    # 处理 /tmp/openssl.cnf， 替换变量值
    # 拷贝 /vagrant/tls/master-kubeconfig.yaml 到 /etc/kubernetes/master-kubeconfig.yaml
  else
    # 拷贝 tls/make-certs-node.sh 到 /tmp/make-certs.sh
    # 拷贝 tls/openssl-node.cnf.tmpl 到 /tmp/openssl.cnf
    # 拷贝 /vagrant/tls/node-kubeconfig.yaml 到 /etc/kubernetes/node-kubeconfig.yaml
    # 处理 /tmp/openssl.cnf， 替换变量值
    # 处理 /etc/kubernetes/node-kubeconfig.yaml， 替换变量值
  end

  # Process Kubernetes manifests, depending on node type
  # ## [provision] 处理 Kubernetes manifests
  begin
    if vmName == "master"
      # ## 拷贝 /vagrant/manifests/master* 到 /etc/kubernetes/manifests 
    else
      # ## 拷贝 /vagrant/manifests/node* 到 /etc/kubernetes/manifests 
    end
    # ## 处理 /vagrant/manifests/*， 替换变量值
  end

  # Process vagrantfile
  # ## [provision] Process vagrantfile
  if File.exist?(cfg)
    # ## cfg 为 master.yaml 或 node.yml
    # ## cfg 拷贝到 /tmp/vagrantfile-user-data
    # ## 处理 /tmp/vagrantfile-user-data， 替换变量值
    # ## 将 /tmp/vagrantfile-user-data 拷贝到 /var/lib/coreos-vagrant/vagrantfile-user-data
  end

end

```

###### 分阶段文件检查
1. before [:up, :provision]
  `temp/setup` (from `setup.tmpl`)
  `temp/coredns-deployment.yaml` (from `plugins/dns/coredns.yaml.sed`)
2. provision (create setup / dns file)
  (src：`setup.tmpl` -> `temp/setup` -> `/home/core/kubectlsetup`)
  [Coredns]
  `/home/core/coredns-deployment.yaml` (from `temp/coredns-deployment.yaml`)
  [kube-dns]
  `/home/core/dns-configmap.yaml` (from `plugins/dns/kube-dns/dns-configmap.yaml`)
  `/home/core/dns-deployment.yaml` (from `temp/dns-deployment.yaml`)
  `/home/core/dns-service.yaml` (from `plugins/dns/kube-dns/dns-service.yaml`)
  `/home/core/dns-service-rbac.yaml` (from `plugins/dns/kube-dns/dns-service-rbac.yaml`)
 	[USE_KUBE_UI]
  `/home/core/dashboard-deployment.yaml` (from `plugins/dashboard/dashboard-deployment.yaml`)
  `/home/core/dashboard-service.yaml` (from `plugins/dashboard/dashboard-service.yaml`) 
3. provision (Copy TLS stuff)
  `/tmp/make-certs.sh` (from `tls/make-certs-master.sh`)
  `/tmp/openssl.cnf` (from `openssl-master.cnf.tmpl`)
  `/etc/kubernetes/master-kubeconfig.yaml` (from `tls/master-kubeconfig.yaml`)
4. Process Kubernetes manifests, depending on node type
  Master
  `/etc/kubernetes/manifests/master-apiserver-rbac.yaml`
  `/etc/kubernetes/manifests/master-apiserver.yaml`
  `/etc/kubernetes/manifests/master-controller-manager.yaml`
  `/etc/kubernetes/manifests/master-proxy.yaml`
  `/etc/kubernetes/manifests/master-scheduler.yaml`
  Node
  `/etc/kubernetes/manifests/node-proxy.yaml`
5. provision (Process vagrantfile)
  `/var/lib/coreos-vagrant/vagrantfile-user-data`
  Master: (src `master.yaml` -> `/var/lib/coreos-vagrant/vagrantfile-user-data`)  
  node: (src `node.yaml` -> `/var/lib/coreos-vagrant/vagrantfile-user-data`)  
6. after [:up, :resume] 
  `/home/core/kubectlsetup install`


---
###### vagrantfile-user-data 
文件结构
```yaml
#cloud-config
---
write-files
  - path: /etc/conf.d/nfs …
  - path: /opt/bin/wupiao …
coreos:
  flannel:
    interface: $public_ipv4
  units:
    - name: rpcbind.service ...
    - name: rpc-statd.service ...
    - name: etcd-member.service ...
    - name: flanneld.service ...
    - name: docker.service ...
    - name: early-docker.service ...
    - name: kube-certs.service ...
    - name: hyperkube-download.service ...
    - name: hyperkube-import.service ...
    - name: kube-kubelet.service ...
  update:
    group: alpha
    reboot-strategy: off
```

* `rpcbind.service`  {enable: true, command: start }
  [active] [running]
* `rpc-statd.service` {enable: true, command: start }
  [active] [running]
* `etcd-member.service` {Type=notify, Restart=always ExecStart=...}
  [active] [running]
* `flanneld.service` { command: start }
  [active] [running]
* `docker.service` { command: start }
  [active] [running]
* `early-docker.service`
  [--]
* `kube-certs.service`  
  { `command: start ExecStart=/tmp/make-certs.sh Type=oneshot RemainAfterExit=true` }
  [active] [dead]
* `hyperkube-download.service` { Type=oneshot RemainAfterExit=true } 
  `ExecStart=/usr/bin/docker pull gcr.io/google_containers/hyperkube-amd64:v1.9.2`
  `ExecStart=/usr/bin/docker save --output /vagrant/artifacts/hyperkube_v1.9.2.tar gcr.io/google_containers/hyperkube-amd64:v1.9.2`
  [active] [exited]
* `hyperkube-import.service` { command: start Type=oneshot RemainAfterExit=true }
  `ExecStart=/usr/bin/docker load --input /vagrant/artifacts/hyperkube_v1.9.2.tar`
  [inactive] [dead]
* `kube-kubelet.service` { command: start Restart=on-failure ExecStart=...}
  [inactive] [dead]


master.yaml(vagrantfile-user-data)
```
rpcbind.service              loaded active running RPC Bind
rpc-statd.service            loaded active running NFS status monitor for NFSv2/3 locking.
etcd-member.service          loaded active running etcd
flanneld.service             loaded active running flannel - Network fabric for containers (System Application Container)
docker.service               loaded active running Docker Application Container Engine
early-docker.service
kube-certs.service
hyperkube-download.service   loaded active exited  Download Hyperkube Docker image
hyperkube-import.service
kube-kubelet.service
```

###### after [:up, :resume] 
```ruby
kHost.trigger.after [:up] do
  info "Waiting for Kubernetes master to become ready..."
  j, uri, res = 0, URI("http://#{MASTER_IP}:8080"), nil
  loop do
    j += 1
    begin
      res = Net::HTTP.get_response(uri)
    rescue
      sleep 10
    end
    break if res.is_a? Net::HTTPSuccess or j >= BOX_TIMEOUT_COUNT
  end
  if res.is_a? Net::HTTPSuccess
    info "#{Time.now}: successfully deployed #{vmName}"
  else
    info "#{Time.now}: failed to deploy #{vmName} within timeout count of #{BOX_TIMEOUT_COUNT}"
  end

  info "Installing kubectl for the Kubernetes version we just bootstrapped..."
  if OS.windows?
    run_remote "sudo -u core /bin/sh /home/core/kubectlsetup install"
  else
    system "./temp/setup install"
  end

  # set cluster
  if OS.windows?
    run_remote "/opt/bin/kubectl config set-cluster default-cluster --server=https://#{MASTER_IP} --certificate-authority=/vagrant/artifacts/tls/ca.pem"
    run_remote "/opt/bin/kubectl config set-credentials default-admin --certificate-authority=/vagrant/artifacts/tls/ca.pem --client-key=/vagrant/artifacts/tls/admin-key.pem --client-certificate=/vagrant/artifacts/tls/admin.pem"
    run_remote "/opt/bin/kubectl config set-context local --cluster=default-cluster --user=default-admin"
    run_remote "/opt/bin/kubectl config use-context local"
  else
    system "kubectl config set-cluster default-cluster --server=https://#{MASTER_IP} --certificate-authority=artifacts/tls/ca.pem"
    system "kubectl config set-credentials default-admin --certificate-authority=artifacts/tls/ca.pem --client-key=artifacts/tls/admin-key.pem --client-certificate=artifacts/tls/admin.pem"
    system "kubectl config set-context local --cluster=default-cluster --user=default-admin"
    system "kubectl config use-context local"
  end

  info "Configuring Kubernetes DNS..."

  if DNS_PROVIDER == "kube-dns"
      res, uri.path = nil, '/api/v1/namespaces/kube-system/replicationcontrollers/kube-dns'
      begin
        res = Net::HTTP.get_response(uri)
      rescue
      end
      if not res.is_a? Net::HTTPSuccess
        if OS.windows?
          run_remote "/opt/bin/kubectl create -f /home/core/dns-deployment.yaml"
        else
          system "kubectl create -f temp/dns-deployment.yaml"
        end
      end

      res, uri.path = nil, '/api/v1/namespaces/kube-system/services/kube-dns'
      begin
        res = Net::HTTP.get_response(uri)
      rescue
      end
      if not res.is_a? Net::HTTPSuccess
        if OS.windows?
          run_remote "/opt/bin/kubectl create -f /home/core/dns-configmap.yaml"
          run_remote "/opt/bin/kubectl create -f /home/core/dns-service.yaml"
        else
          system "kubectl create -f plugins/dns/kube-dns/dns-configmap.yaml"
          # Use service file specific to RBAC
          if AUTHORIZATION_MODE == "RBAC"
            system "kubectl create -f plugins/dns/kube-dns/dns-service-rbac.yaml"
          else
            system "kubectl create -f plugins/dns/kube-dns/dns-service.yaml"
          end
        end
      end
  else if DNS_PROVIDER == "coredns"
          res, uri.path = nil, '/api/v1/namespaces/kube-system/deployment/coredns'
          begin
            res = Net::HTTP.get_response(uri)
          rescue
          end
          if not res.is_a? Net::HTTPSuccess
            if OS.windows?
              run_remote "/opt/bin/kubectl create -f /home/core/coredns-deployment.yaml"
            else
              system "kubectl create -f temp/coredns-deployment.yaml"
            end
          end
        end
  end

  if USE_KUBE_UI
    info "Configuring Kubernetes dashboard..."

    res, uri.path = nil, '/api/v1/namespaces/kube-system/replicationcontrollers/kubernetes-dashboard'
    begin
      res = Net::HTTP.get_response(uri)
    rescue
    end
    if not res.is_a? Net::HTTPSuccess
      if OS.windows?
        run_remote "/opt/bin/kubectl create -f /home/core/dashboard-deployment.yaml"
      else
        system "kubectl create -f plugins/dashboard/dashboard-deployment.yaml"
      end
    end

    res, uri.path = nil, '/api/v1/namespaces/kube-system/services/kubernetes-dashboard'
    begin
      res = Net::HTTP.get_response(uri)
    rescue
    end
    if not res.is_a? Net::HTTPSuccess
      if OS.windows?
        run_remote "/opt/bin/kubectl create -f /home/core/dashboard-service.yaml"
      else
        system "kubectl create -f plugins/dashboard/dashboard-service.yaml"
      end
    end

    info "Kubernetes dashboard will be available at http://#{MASTER_IP}:8080/ui"
  end

end
```

trigger.after [:up] 
```bash
# Installing kubectl for the Kubernetes version we just bootstrapped
sudo -u core /bin/sh /home/core/kubectlsetup install
# set cluster
/opt/bin/kubectl config set-cluster default-cluster --server=https://192.168.99.101 --certificate-authority=/vagrant/artifacts/tls/ca.pem
/opt/bin/kubectl config set-credentials default-admin --certificate-authority=/vagrant/artifacts/tls/ca.pem --client-key=/vagrant/artifacts/tls/admin-key.pem --client-certificate=/vagrant/artifacts/tls/admin.pem
/opt/bin/kubectl config set-context local --cluster=default-cluster --user=default-admin
/opt/bin/kubectl config use-context local
# Configuring Kubernetes DNS...
/opt/bin/kubectl create -f /home/core/coredns-deployment.yaml
```


-----
### CoreOS File List
* `~/coredns-deployment.yaml`
* `~/kubectlsetup`
* `/etc/kubernetes/master-kubeconfig.yaml`
* `/etc/kubernetes/manifests/master-apiserver-rbac.yaml`
* `/etc/kubernetes/manifests/master-apiserver.yaml`
* `/etc/kubernetes/manifests/master-controller-manager.yaml`
* `/etc/kubernetes/manifests/master-proxy.yaml`
* `/etc/kubernetes/manifests/master-scheduler.yaml`
* `/var/lib/coreos-vagrant/vagrantfile-user-data`
* `/tmp/make-certs.sh`
* `/tmp/openssl.cnf`


###### coredns-deployment.yaml
@import "bak/coredns-deployment.yaml"

###### kubectlsetup
@import "bak/kubectlsetup" {as=bash}

###### master-kubeconfig.yaml
@import "bak/master-kubeconfig.yaml"

###### master-apiserver-rbac.yaml
@import "bak/master-apiserver-rbac.yaml"

###### master-apiserver.yaml
@import "bak/master-apiserver.yaml"

###### master-controller-manager.yaml
@import "bak/master-controller-manager.yaml"

###### master-proxy.yaml
@import "bak/master-proxy.yaml"

###### master-scheduler.yaml
@import "bak/master-scheduler.yaml"

###### vagrantfile-user-data
@import "bak/vagrantfile-user-data" {as=yaml}


###### make-certs.sh
@import "bak/make-certs.sh"

###### openssl.cnf
@import "bak/openssl.cnf" {as=yaml}



---
### 文件清单（宿主机）
###### Vagrantfile
@import "Vagrantfile" {as=ruby}

###### master.yaml
@import "master.yaml"

###### node.yaml
@import "node.yaml"

###### setup.tmpl
@import "setup.tmpl" {as=bash}