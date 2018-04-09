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
  - https://{{ inventory_hostname }}.na.domain.com:2379
  - https://{{ master0 }}.domain.com:2379
  - https://{{ master1 }}.na.domain.com:2379
 ``` 
