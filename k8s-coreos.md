## kubernetes-vagrant-coreos-cluster
Kubernetes cluster (for testing purposes) made easy with Vagrant and CoreOS.
参考 [pires/kubernetes-vagrant-coreos-cluster](https://github.com/pires/kubernetes-vagrant-coreos-cluster).


### 修改说明
###### Update Log
Update 1: 为网路环境修改
* 新增说明文档： `k8s-coreos.md`、`k8s-coreos.html`
* 修改 `Vagrantfile`, 修改 `MASTER_CPUS` / `BASE_IP_ADDR` 设置 与 windows 重定向问题
* 修改 `plugins/dns/coredns/deploy.sh`, 解决 windows 下脚本重定向输出问题（结合`Vagrantfile`修改）。
* `setup.tmpl` 文件, 修改 “`KUB_URL=`” 为本地下载地址  
* 以下文件中 “`gcr.io/google_containers/hyperkube-amd64`” 修改为 “`wangwg2/hyperkube-amd64`”
  `master.yaml`
  `manifests/master-apiserver-rbac.yaml`
  `manifests/master-apiserver.yaml`
  `manifests/master-controller-manager.yaml`
  `manifests/master-proxy.yaml`
  `manifests/master-scheduler.yaml`
  `manifests/node-proxy.yaml`
* `hyperkube kubelet` 会 pull 镜像 `gcr.io/google_containers/pause-amd64:3.0`。
  解决镜像被墙问题，改为 `wangwg2/pause-amd64:3.0`。
  为 `\hyperkube kubelet` 增加 `--pod-infra-container-image=wangwg2/pause-amd64:3.0 \`
* `docker logs -f container_name` 查看容器日志分析问题。


###### Vagrantfile 解析
---
预备
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


---
###### Vagrantfile 解析 - 核心部分结构
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

---
###### 修改文件列表
* `setup.tmpl`
  解决 `bukectl`下载慢问题。预先下载
* `Vagrantfile`
  Vagrant 虚机 CPU / IP 设置与 windows 重定向问题。
* `plugins/dns/coredns/deploy.sh`
  Windows 重定向问题。


```
master.yaml
kubernetes-coreos.md
kubernetes-coreos-docs.md
Vagrantfile
plugins/dns/coredns/coredns-deployment.yaml
plugins/dns/coredns/deploy.sh
plugins/dns/kube-dns/dns-deployment-rbac.yaml.tmpl
plugins/dns/kube-dns/dns-deployment.yaml.tmpl
```


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

---
@import "setup.tmpl"


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

* rpcbind.service  {enable: true, command: start }
  [active] [running]
* rpc-statd.service {enable: true, command: start }
  [active] [running]
* etcd-member.service {Type=notify, Restart=always ExecStart=...}
  [active] [running]
* flanneld.service { command: start }
  [active] [running]
* docker.service { command: start }
  [active] [running]
* early-docker.service
  [--]
* kube-certs.service  
  { command: start ExecStart=/tmp/make-certs.sh Type=oneshot RemainAfterExit=true }
  [active] [dead]
* hyperkube-download.service { Type=oneshot RemainAfterExit=true } 
  `ExecStart=/usr/bin/docker pull gcr.io/google_containers/hyperkube-amd64:v1.9.2`
  `ExecStart=/usr/bin/docker save --output /vagrant/artifacts/hyperkube_v1.9.2.tar gcr.io/google_containers/hyperkube-amd64:v1.9.2`
  [active] [exited]
* hyperkube-import.service { command: start Type=oneshot RemainAfterExit=true }
  `ExecStart=/usr/bin/docker load --input /vagrant/artifacts/hyperkube_v1.9.2.tar`
  [inactive] [dead]
* kube-kubelet.service { command: start Restart=on-failure ExecStart=...}
  [inactive] [dead]


内容
```yaml
#cloud-config

---
write-files:
  - path: /etc/conf.d/nfs
    permissions: '0644'
    content: |
      OPTS_RPC_MOUNTD=""
  - path: /opt/bin/wupiao
    permissions: '0755'
    content: |
      #!/bin/bash
      # [w]ait [u]ntil [p]ort [i]s [a]ctually [o]pen
      [ -n "$1" ] && \
        until curl -o /dev/null -sIf http://${1}; do \
          sleep 1 && echo .;
        done;
      exit $?

coreos:
  flannel:
    interface: $public_ipv4
  units:
    - name: rpcbind.service
      enable: true
      command: start
    - name: rpc-statd.service
      enable: true
      command: start
    - name: etcd-member.service
      command: start
      content: |
        [Unit]
        Description=etcd
        Documentation=https://github.com/coreos/etcd
        [Service]
        Environment='ETCD_IMAGE_TAG=v3.1.8'
        Environment='ETCD_DATA_DIR=/var/lib/etcd'
        Environment='ETCD_USER=etcd'
        Type=notify
        Restart=always
        RestartSec=5s
        LimitNOFILE=40000
        TimeoutStartSec=0
        ExecStart=/usr/lib/coreos/etcd-wrapper --name master \
            --listen-client-urls http://0.0.0.0:2379,http://0.0.0.0:4001 \
            --advertise-client-urls http://$public_ipv4:2379,http://$public_ipv4:4001 \
            --listen-peer-urls http://$private_ipv4:2380,http://$private_ipv4:7001 \
            --initial-advertise-peer-urls http://$private_ipv4:2380 \
            --initial-cluster master=http://192.168.99.101:2380
        [Install]
        WantedBy=multi-user.target
    - name: flanneld.service
      command: start
      drop-ins:
        - name: 50-network-config.conf
          content: |
            [Unit]
            Requires=etcd-member.service
            [Service]
            ExecStartPre=/usr/bin/etcdctl set /coreos.com/network/config '{"Network":"10.244.0.0/16", "Backend": {"Type": "host-gw"}}'
    - name: docker.service
      command: start
      drop-ins:
        - name: 51-docker-mirror.conf
        - name: 40-flannel.conf
          content: |
            [Unit]
            Requires=flanneld.service
            After=flanneld.service
        - name: 50-docker-options.conf
          content: |
            [Service]
            Environment='DOCKER_OPTS=--storage-driver=overlay2 '
    - name: early-docker.service
      drop-ins:
    - name: kube-certs.service
      command: start
      content: |
        [Unit]
        Description=Generate Kubernetes API Server certificates
        ConditionPathExists=/tmp/make-certs.sh
        Requires=network-online.target
        After=network-online.target
        [Service]
        ExecStartPre=-/usr/sbin/groupadd -r kube-cert
        ExecStartPre=/usr/bin/chmod 755 /tmp/make-certs.sh
        ExecStart=/tmp/make-certs.sh
        Type=oneshot
        RemainAfterExit=true
    - name: hyperkube-download.service
      command: start
      content: |
        [Unit]
        Description=Download Hyperkube Docker image
        Requires=docker.service
        After=docker.service
        ConditionPathExists=!/vagrant/artifacts/hyperkube_v1.9.2.tar
        [Service]
        ExecStart=/usr/bin/docker pull gcr.io/google_containers/hyperkube-amd64:v1.9.2
        ExecStart=/usr/bin/docker save --output /vagrant/artifacts/hyperkube_v1.9.2.tar gcr.io/google_containers/hyperkube-amd64:v1.9.2
        Type=oneshot
        RemainAfterExit=true
    - name: hyperkube-import.service
      command: start
      content: |
        [Unit]
        Description=Import Hyperkube Docker image
        Requires=docker.service
        After=docker.service
        ConditionPathExists=/vagrant/artifacts/hyperkube_v1.9.2.tar
        [Service]
        ExecStart=/usr/bin/docker load --input /vagrant/artifacts/hyperkube_v1.9.2.tar
        Type=oneshot
        RemainAfterExit=true
    - name: kube-kubelet.service
      command: start
      content: |
        [Unit]
        Description=Kubernetes Kubelet
        Documentation=https://github.com/GoogleCloudPlatform/kubernetes
        Requires=kube-certs.service
        Wants=hyperkube-download.service hyperkube-import.service
        After=kube-certs.service hyperkube-download.service hyperkube-import.service
        [Service]
        ExecStartPre=/usr/bin/mkdir -p /etc/kubernetes/manifests
        ExecStartPre=-/usr/bin/docker rm -f kubelet
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
          gcr.io/google_containers/hyperkube-amd64:v1.9.2 \
            /hyperkube kubelet \
            --address=$private_ipv4 \
            --network-plugin= \
            --register-schedulable=false \
            --allow-privileged=true \
            --pod-manifest-path=/etc/kubernetes/manifests \
            --hostname-override=$public_ipv4 \
            --cluster_dns=10.100.0.10 \
            --cluster_domain=cluster.local \
            --kubeconfig=/etc/kubernetes/master-kubeconfig.yaml
        Restart=on-failure
        RestartSec=10
        WorkingDirectory=/root/
        [Install]
        WantedBy=multi-user.target
  update:
    group: alpha
    reboot-strategy: off
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

---
###### kubectl 下载慢 （`setup.tmpl`）
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

###### Vagrant 虚机设置（`Vagrantfile`）
```ruby
# 解决CPU不足
MASTER_CPUS = ENV['MASTER_CPUS'] || 1
NODE_CPUS = ENV['NODE_CPUS'] || 1
# IP网络段
BASE_IP_ADDR = ENV['BASE_IP_ADDR'] || "192.168.99"
```

###### DOCKER_OPTIONS（`Vagrantfile`）
```ruby
DOCKER_OPTIONS = ENV['DOCKER_OPTIONS'] || '--registry-mirror=https://4ue5z1dy.mirror.aliyuncs.com'
```

----
###### Window shell 重定向问题 （`Vagrantfile` + `plugins/dns/coredns/deploy.sh`）
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
      system "#{__dir__}/plugins/dns/coredns/deploy.sh 10.100.0.10/24 #{DNS_DOMAIN} #{__dir__}/temp/coredns-deployment.yaml"
    end
  end
```

`plugins/dns/coredns/deploy.sh`
```bash
#!/bin/bash

# Deploys CoreDNS to a cluster currently running Kube-DNS.
# --- 修改前 ---
SERVICE_CIDR=$1
CLUSTER_DOMAIN=${2:-cluster.local}
YAML_TEMPLATE=${3:-`pwd`/coredns.yaml.sed}
YAML=${4:-`pwd`/coredns.yaml}

# --- 修改后 ---
SERVICE_CIDR=$1
CLUSTER_DOMAIN=${2:-cluster.local}
YAML=${3:-`pwd`/coredns.yaml}
# --------------

if [[ -z $SERVICE_CIDR ]]; then
	echo "Usage: $0 SERVICE-CIDR [ CLUSTER-DOMAIN ] [ YAML-TEMPLATE ] [ YAML ]"
	exit 1
fi

CLUSTER_DNS_IP="10.100.0.10"
# --- 修改前 ---
sed -e s/CLUSTER_DNS_IP/$CLUSTER_DNS_IP/g -e s/CLUSTER_DOMAIN/$CLUSTER_DOMAIN/g -e s?SERVICE_CIDR?$SERVICE_CIDR?g $YAML_TEMPLATE
# --- 修改后 ---
sed -i s/CLUSTER_DNS_IP/$CLUSTER_DNS_IP/g -e s/CLUSTER_DOMAIN/$CLUSTER_DOMAIN/g -e s?SERVICE_CIDR?$SERVICE_CIDR?g $YAML
```


-----
### 文件

###### Vagrantfile
@import "Vagrantfile" {as=ruby}

###### bak/coredns-deployment.yaml
@import "bak/coredns-deployment.yaml"

###### bak/kubectlsetup
@import "bak/kubectlsetup" {as=bash}

###### bak/etc/kubernetes/master-kubeconfig.yaml
@import "bak/etc/kubernetes/master-kubeconfig.yaml"

###### bak/etc/kubernetes/master-kubeconfig.yaml
@import "bak/etc/kubernetes/manifests/master-apiserver-rbac.yaml"
###### bak/etc/kubernetes/master-apiserver.yaml
@import "bak/etc/kubernetes/manifests/master-.yaml"
###### bak/etc/kubernetes/master-controller-manager.yaml
@import "bak/etc/kubernetes/manifests/master-controller-manager.yaml"
###### bak/etc/kubernetes/master-proxy.yaml
@import "bak/etc/kubernetes/manifests/master-.yaml"
###### bak/etc/kubernetes/master-scheduler.yaml
@import "bak/etc/kubernetes/manifests/master-scheduler.yaml"

###### bak/var/lib/coreos-vagrant/vagrantfile-user-data
@import "bak/var/lib/coreos-vagrant/vagrantfile-user-data" {as=yaml}

###### bak/tmp/make-certs.sh
@import "bak/tmp/make-certs.sh" 
###### bak/tmp/openssl.cnf
@import "bak/tmp/openssl.cnf" {as=ini}
