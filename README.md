# openshift-random

#### Node Preparation
```
- name: validate default trust policy
  command: atomic trust show
  register: atomic

- debug:
    msg: "trust default set to accept-all"
  when: atomic.stdout | search("accept")

- name: Add Red Hat to trusted registries
  command: >
    atomic trust add registry.access.redhat.com
    --pubkeys /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
    --sigstore https://access.redhat.com/webassets/docker/content/sigstore
    --type signedBy
```
#### We now converted to overlay2
```
- name: update docker-storage-setup configuration
  blockinfile:
    state: present
    dest: /etc/sysconfig/docker-storage-setup
    content: |
        STORAGE_DRIVER=overlay2
  tags:
  - docker
```  

##### We enforce an empty dir quota for OCP >= 3.7

```  
  - name: create /var/lib/origin/openshift.local.volumes
  file:
    owner: root
    group: root
    path: /var/lib/origin/openshift.local.volumes
    state: directory
    setype: docker_var_lib_t
  tags:
  - storage

- name: format disk for var_lib_origin_local_volumes
  filesystem:
    fstype: xfs
    dev: /dev/{{ volume_group }}/{{ empty_dir }}
  tags:
  - storage

- name: mount /var/lib/origin.local.vols
  mount:
    name: /var/lib/origin/openshift.local.volumes
    src: /dev/mapper/{{ volume_group }}-{{ empty_dir }}
    opts: defaults,norelatime,logbufs=8,logbsize=256k,largeio,inode64,swalloc,allocsize=2M,gquota
    fstype: xfs
    state: mounted
  tags:
  - storage
```

#### Deploy master configuration based off jinja / env including:
```
  - name: deploy master-configuration
  template:
    src: master-config.yaml.j2
    dest: /etc/origin/master/master-config.yaml
    owner: root
    group: root
    backup: yes
    
            kind: BuildDefaultsConfig
        resources:
          limits:
            cpu: 100m
            memory: 256m
          requests:
            cpu: 100m
            memory: 256m
            
            etcdClientInfo:
  ca: master.etcd-ca.crt
  certFile: master.etcd-client.crt
  keyFile: master.etcd-client.key
  urls:
  - https://{{ inventory_hostname }}.domain.com:2379
  - https://{{ master0 }}.domain.com:2379
  - https://{{ master1 }}.domain.com:2379
 ``` 
#### Deploy node configuration based off jinja / env including:

```

apiVersion: v1
dnsBindAddress: 127.0.0.1:53
dnsRecursiveResolvConf: /etc/origin/node/resolv.conf
dnsDomain: cluster.local
dnsIP: {{ hostvars[inventory_hostname]['ansible_default_ipv4']['address'] }}
dockerConfig:
  execHandlerName: ""
iptablesSyncPeriod: "30s"
imageConfig:
  format: openshift3/ose-${component}:${version}
  latest: false
kind: NodeConfig
kubeletArguments:
  enable-controller-attach-detach:
  - 'true'
  cpu-cfs-quota:
  - "false"
  eviction-hard:
  - "memory.available<1Gi"
  image-gc-high-threshold:
  - "80"
  image-gc-low-threshold:
  - "60"
  kube-reserved:
  - "cpu={{ ((ansible_processor_vcpus * 100) * 1) }}m,memory={{ ansible_memtotal_mb * 0.05 }}Mi"
  system-reserved:
  - "cpu={{ ((ansible_processor_vcpus * 100) * 1) }}m,memory={{ ansible_memtotal_mb * 0.10 }}Mi"
  max-pods:
  - "250"
  serialize-image-pulls:
  - "false"
  node-labels:
  - datacenter=<something>
  - region=<something>
  - zone=<something>
  - network_zone=<something>
  maximum-dead-containers-per-container:
  - "2"
  minimum-container-ttl-duration:
  - "1m"
  maximum-dead-containers:
  - "325"
masterClientConnectionOverrides:
  acceptContentTypes: application/vnd.kubernetes.protobuf,application/json
  contentType: application/vnd.kubernetes.protobuf
  burst: 200
  qps: 100
masterKubeConfig: system:node:{{ inventory_hostname }}.domain.com.kubeconfig
networkPluginName: redhat/openshift-ovs-multitenant
# networkConfig struct introduced in origin 1.0.6 and OSE 3.0.2 which
# deprecates networkPluginName above. The two should match.
networkConfig:
   mtu: 1450
   networkPluginName: redhat/openshift-ovs-multitenant
nodeName: {{ inventory_hostname }}.domain.com
podManifestConfig:
servingInfo:
  bindAddress: 0.0.0.0:10250
  certFile: server.crt
  clientCA: ca.crt
  keyFile: server.key
volumeDirectory: /var/lib/origin/openshift.local.volumes
proxyArguments:
  proxy-mode:
     - iptables
volumeConfig:
  localQuota:
    perFSGroup: 512Mi
volumeDirectory: /var/lib/origin/openshift.local.volumes
```

#### OpenShift inventory host examples
```
3.9 = https://github.com/openshift/openshift-ansible/blob/release-3.9/inventory/hosts.example
3.10 = https://github.com/openshift/openshift-ansible/blob/release-3.10/inventory/hosts.example
3.11 = https://github.com/openshift/openshift-ansible/blob/release-3.11/inventory/hosts.example
```
