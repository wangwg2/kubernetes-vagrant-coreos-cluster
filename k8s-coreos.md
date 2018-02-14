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
解决镜像被墙问题，改为 wangwg2/pause-amd64:3.0。
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


---
### Vagrantfile 解析
###### 预备
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

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

###### 核心部分结构
```ruby
config.vm.define vmName = hostname do |kHost|
  kHost.vm.hostname = vmName

  kHost.trigger.reject [:suspend, :resume] do
    info "'vagrant suspend' and 'vagrant resume' are disabled."
  end

  config.trigger.instead_of :reload do
    exec "vagrant halt && vagrant up"
  end

  if vmName == "master"
    kHost.trigger.before [:up, :provision] do
      # create setup file
      if DNS_PROVIDER == "kube-dns"
          # create dns-deployment.yaml file
      else if DNS_PROVIDER == "coredns"
          # create coredns-deployment.yaml file
      end
    end

    kHost.trigger.after [:up] do
      info "Waiting for Kubernetes master to become ready..."
      info "Installing kubectl for the Kubernetes version we just bootstrapped..."
      # run_remote "sudo -u core /bin/sh /home/core/kubectlsetup install"

      # set cluster
      ##  run_remote "/opt/bin/kubectl config **"

      info "Configuring Kubernetes DNS..."
      # if DNS_PROVIDER == "kube-dns"
      #   run_remote "/opt/bin/kubectl create -f /home/core/dns-deployment.yaml"
      #   run_remote "/opt/bin/kubectl create -f /home/core/dns-configmap.yaml"
      #   run_remote "/opt/bin/kubectl create -f /home/core/dns-service.yaml"
      # else if DNS_PROVIDER == "coredns"
      #   run_remote "/opt/bin/kubectl create -f /home/core/coredns-deployment.yaml"
      # end

      # if USE_KUBE_UI
      #   info "Configuring Kubernetes dashboard..."
      #   run_remote "/opt/bin/kubectl create -f /home/core/dashboard-deployment.yaml"
      #   run_remote "/opt/bin/kubectl create -f /home/core/dashboard-service.yaml"
      #   info "Kubernetes dashboard will be available at http://#{MASTER_IP}:8080/ui"
      # end
    end

    # copy setup files to master vm if host is windows
    if OS.windows?
      kHost.vm.provision :file, :source => File.join(File.dirname(__FILE__), "temp/setup"), :destination => "/home/core/kubectlsetup"

      if DNS_PROVIDER == "kube-dns"
        # "plugins/dns/kube-dns/dns-configmap.yaml" => "/home/core/dns-configmap.yaml"
        # "temp/dns-deployment.yaml"                => "/home/core/dns-deployment.yaml"
        # "plugins/dns/kube-dns/dns-service.yaml"   => "/home/core/dns-service.yaml"
        # "plugins/dns/kube-dns/dns-service-rbac.yaml"  => "/home/core/dns-service-rbac.yaml"
      else if DNS_PROVIDER == "coredns"
        # "temp/coredns-deployment.yaml"            => "/home/core/coredns-deployment.yaml"
      end

      if USE_KUBE_UI
        # "plugins/dashboard/dashboard-deployment.yaml" => "/home/core/dashboard-deployment.yaml"
        # "plugins/dashboard/dashboard-service.yaml"    => "/home/core/dashboard-service.yaml"
      end
    end

    # clean temp directory after master is destroyed
    kHost.trigger.after [:destroy] do
    end
  end

  if vmName == "node-%02d" % (i - 1)
    kHost.trigger.before [:up, :provision] do
      info "#{Time.now}: setting up node..."
    end

    kHost.trigger.after [:up] do
      info "Waiting for Kubernetes minion [node-%02d" % (i - 1) + "] to become ready..."
      if hasResponse
        info "#{Time.now}: successfully deployed #{vmName}"
      else
        info "#{Time.now}: failed to deploy #{vmName} within timeout count of #{BOX_TIMEOUT_COUNT}"
      end
    end
  end

  kHost.trigger.before [:halt, :reload] do
    if REMOVE_VAGRANTFILE_USER_DATA_BEFORE_HALT
      run_remote "sudo rm -f /var/lib/coreos-vagrant/vagrantfile-user-data"
    end
  end

  if SERIAL_LOGGING
  end

  kHost.vm.network :private_network, ip: "#{BASE_IP_ADDR}.#{i+100}"

  # you can override this in synced_folders.yaml
  kHost.vm.synced_folder ".", "/vagrant", disabled: true

  # /home/core/.dockercfg 
  # /root/.dockercfg
  if USE_DOCKERCFG && File.exist?(DOCKERCFG)
  end

  # Copy TLS stuff
    # tls/make-certs-master.sh
    # /tmp/make-certs.sh
    # tls/openssl-master.cnf.tmpl
    # /tmp/openssl.cnf
    # /vagrant/tls/master-kubeconfig.yaml
    # /etc/kubernetes/master-kubeconfig.yaml

  # Process Kubernetes manifests, depending on node type
  ## 目标路径： /etc/kubernetes/manifests/
  ## master: 
      # manifests/master-apiserver-rbac.yaml
      # manifests/master-apiserver.yaml
      # manifests/master-controller-manager.yaml
      # manifests/master-proxy.yaml
      # manifests/master-scheduler.yaml
  ## node: 
      # manifests/node-proxy.yaml
  begin
    if vmName == "master"
    else
    end
  end

  # Process vagrantfile (provision)
  ## cfg 为 master.yaml 或 node.yml
  ## master:  master.yaml -> /var/lib/coreos-vagrant/vagrantfile-user-data
  ## node:    node.yaml   -> /var/lib/coreos-vagrant/vagrantfile-user-data
  if File.exist?(cfg)
  end

end
```

###### 分阶段文件检查
1. before [:up, :provision]
  `temp/setup`
  temp/coredns-deployment.yaml
2. provision (create setup / dns file)
  `/home/core/kubectlsetup` 
  (src：setup.tmpl -> temp/setup -> /home/core/kubectlsetup)
  [Coredns]
  `/home/core/coredns-deployment.yaml`
  [kube-dns]
  `/home/core/dns-configmap.yaml`
  `/home/core/dns-deployment.yaml`
  `/home/core/dns-service.yaml`
  `/home/core/dns-service-rbac.yaml`
 	[USE_KUBE_UI]
  `/home/core/dashboard-deployment.yaml`
  `/home/core/dashboard-service.yaml`      
3. provision (Copy TLS stuff)
  `/tmp/make-certs.sh`
  `/tmp/openssl.cnf`
  `/etc/kubernetes/master-kubeconfig.yaml`
4. provision (Process vagrantfile)
  `/var/lib/coreos-vagrant/vagrantfile-user-data`
  (src master.yaml -> /var/lib/coreos-vagrant/vagrantfile-user-data)  
5. after [:up, :resume] 
  `/home/core/kubectlsetup install`

---
###### docker images
```
manifests/master-apiserver-rbac.yaml
manifests/master-apiserver.yaml
manifests/master-controller-manager.yaml
manifests/master-proxy.yaml
manifests/master-scheduler.yaml
master.yaml/node.yaml
```

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
###### CoreOS File List
```
~/coredns-deployment.yaml
~/kubectlsetup
/etc/kubernetes/master-kubeconfig.yaml
/etc/kubernetes/manifests/master-apiserver-rbac.yaml
/etc/kubernetes/manifests/master-apiserver.yaml
/etc/kubernetes/manifests/master-controller-manager.yaml
/etc/kubernetes/manifests/master-proxy.yaml
/etc/kubernetes/manifests/master-scheduler.yaml
/var/lib/coreos-vagrant/vagrantfile-user-data
/tmp/make-certs.sh
/tmp/openssl.cnf
```


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