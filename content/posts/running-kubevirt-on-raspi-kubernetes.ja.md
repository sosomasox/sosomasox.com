---
title: "ラズパイKubernetesにKubeVirtを導入してみた"
date: 2022-03-07T18:17:34+09:00
draft: false
Description: ""
thumbnail: /images/running-kubevirt-on-raspi-kubernetes/thumbnail.png
Tags: []
Categories: []
---

&nbsp;

ラズパイk8sでKubeVirt使ってみたいなと思い、導入してみた。

&nbsp;



# 動作環境

[Harvester](https://harvesterhci.io/)に習って、Kubernetes + Longhorn + MinIO の環境を整備。

- Raspberry Pi 4B 8GB USB SSD ブート
- Kubernetes v1.23.1
- containerd v1.5.5
- runc 1.0.1-0ubuntu2~20.04.1
- Longhorn(ストレージクラス)
- MinIO(オブジェクトストレージ)

\**ドキュメントによると、[KubeVirtはコンテナランタイムとしてcontainerd、runcはサポートしていない](http://kubevirt.io/user-guide/operations/installation/#container-runtime-support)が検証としては問題なく動いた。*

&nbsp;



# KubeVirtのインストール
 
基本的に公式ドキュメントの[quickstart](https://kubevirt.io/quickstart_cloud/)をやっていくだけ。

そのままやるとarm64非対応のコンテナイメージが使用されるため、kubevirt-operatorに対してarm64対応のイメージを使うように変更する。

```bash
$ diff kubevirt-operator.yaml.org kubevirt-operator.yaml
6212c6212
<           value: quay.io/kubevirt/virt-operator:v0.50.0
---
>           value: quay.io/kubevirt/virt-operator:20220228_38eb4710e-arm64
6217c6217
<         image: quay.io/kubevirt/virt-operator:v0.50.0
---
>         image: quay.io/kubevirt/virt-operator:20220228_38eb4710e-arm64
```

&nbsp;



# KubeVirtの動作検証

インストールが終わったら、[Labドキュメント](https://kubevirt.io/labs/kubernetes/lab1.html)に従い、KubeVirtを使ってみる。

ここでもひと手間。
仮想ディスクとして使うコンテナイメージをarm64対応のイメージに変える。


```bash
$ diff vm.yaml.org vm.yaml
27c27
<             memory: 64M
---
>             memory: 1024M
34c34
<             image: quay.io/kubevirt/cirros-container-disk-demo
---
>             image: quay.io/kubevirt/cirros-container-disk-demo:20220228_38eb4710e-arm64
```

また、デフォルトで設定されているVMに割り当てるメモリが少なすぎて動作が悪いので増やして実行すると起動した。

![Experiment KubeVirt on raspi kubernetes](/images/running-kubevirt-on-raspi-kubernetes/experiment_kubevirt.png)

&nbsp;



# Containerized Data Importer(CDI)のインストール

Containerized Data Importer(CDI)は、PVを仮想マシンのディスクイメージとして使用できるように、PVCを作成できるようにするアドオン。

ラボドキュメント [Labs - Experiment with CDI](https://kubevirt.io/labs/kubernetes/lab2.html) に従い、インストールする。

例に習い、arm64対応のコンテナイメージを使用するように修正。

```bash
$ diff cdi-operator.yaml.org cdi-operator.yaml
2191a2192,2204
> - apiGroups:
>   - coordination.k8s.io
>   resources:
>   - leases
>   verbs:
>   - '*'
> - apiGroups:
>   - monitoring.coreos.com
>   resources:
>   - prometheusrules
>   - servicemonitors
>   verbs:
>   - '*'
2239c2252
<           value: quay.io/kubevirt/cdi-controller:v1.38.1
---
>           value: quay.io/kubevirt/cdi-controller:20220303_e7e4dfc4-arm64
2241c2254
<           value: quay.io/kubevirt/cdi-importer:v1.38.1
---
>           value: quay.io/kubevirt/cdi-importer:20220303_e7e4dfc4-arm64
2243c2256
<           value: quay.io/kubevirt/cdi-cloner:v1.38.1
---
>           value: quay.io/kubevirt/cdi-cloner:20220303_e7e4dfc4-arm64
2245c2258
<           value: quay.io/kubevirt/cdi-apiserver:v1.38.1
---
>           value: quay.io/kubevirt/cdi-apiserver:20220303_e7e4dfc4-arm64
2247c2260
<           value: quay.io/kubevirt/cdi-uploadserver:v1.38.1
---
>           value: quay.io/kubevirt/cdi-uploadserver:20220303_e7e4dfc4-arm64
2249c2262
<           value: quay.io/kubevirt/cdi-uploadproxy:v1.38.1
---
>           value: quay.io/kubevirt/cdi-uploadproxy:20220303_e7e4dfc4-arm64
2254c2267
<         image: quay.io/kubevirt/cdi-operator:v1.38.1
---
>         image: quay.io/kubevirt/cdi-operator:20220303_e7e4dfc4-arm64
```

Role周りでコケたので、動作するRoleを共有しておく

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    app.kubernetes.io/component: storage
    app.kubernetes.io/managed-by: cdi-operator
    cdi.kubevirt.io: ""
  name: cdi-operator
  namespace: cdi
rules:
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - rolebindings
  - roles
  verbs:
  - '*'
- apiGroups:
  - ""
  resources:
  - serviceaccounts
  - configmaps
  - events
  - secrets
  - services
  verbs:
  - '*'
- apiGroups:
  - apps
  resources:
  - deployments
  - deployments/finalizers
  verbs:
  - '*'
- apiGroups:
  - route.openshift.io
  resources:
  - routes
  - routes/custom-host
  verbs:
  - '*'
- apiGroups:
  - config.openshift.io
  resources:
  - proxies
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - coordination.k8s.io
  resources:
  - leases
  verbs:
  - '*'
- apiGroups:
  - monitoring.coreos.com
  resources:
  - prometheusrules
  - servicemonitors
  verbs:
  - '*'
```

&nbsp;



# Containerized Data Importer(CDI)の動作検証

仮想マシンのディスクイメージを作成するためのPVCを用意。

アノテーション cdi.kubevirt.io/storage.import.endpoint にはクラウドイメージのURLを指定してもいい。
(イメージのダンロードが遅いのか、PVへ書き込みが遅いので自宅環境のオブジェクトストレージ MinIO に保存したイメージを使用した)

![VM images on minio for containerized-data-importer](/images/running-kubevirt-on-raspi-kubernetes/vm_images_on_minio_for_containerized-data-importer.png)

```bash
$ cat pvc_focal-ubuntu.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "focal-ubuntu"
  labels:
    app: containerized-data-importer
  annotations:
    cdi.kubevirt.io/storage.import.endpoint: "http://minio.home.arpa:9000/kubevirt/images/focal-server-cloudimg-arm64.img"
spec:
  storageClassName: longhorn
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 32Gi
```

PVCリソースを作成すると以下のPodが作成される。

```bash
$ kubectl get pod
NAME                         READY   STATUS    RESTARTS   AGE
importer-focal-ubuntu        1/1     Running   0          96s
```

作成されたPodのログをみると、アノテーション cdi.kubevirt.io/storage.import.endpoint で指定したURLからイメージをダンロードし、PVへ書き込みを行っている。

```bash
$ kubectl logs -f importer-focal-ubuntu
I0307 12:03:39.079935       1 importer.go:77] Starting importer
I0307 12:03:39.080680       1 importer.go:144] begin import process
I0307 12:03:39.163006       1 data-processor.go:379] Calculating available size
I0307 12:03:39.163169       1 data-processor.go:391] Checking out file system volume size.
I0307 12:03:39.163229       1 data-processor.go:399] Request image size not empty.
I0307 12:03:39.163298       1 data-processor.go:404] Target size 33484955648.
I0307 12:03:39.163783       1 util.go:39] deleting file: /data/lost+found
I0307 12:03:39.176560       1 nbdkit.go:294] Waiting for nbdkit PID.
I0307 12:03:39.677209       1 nbdkit.go:315] nbdkit ready.
I0307 12:03:39.677326       1 data-processor.go:282] New phase: Convert
I0307 12:03:39.677403       1 data-processor.go:288] Validating image
I0307 12:03:39.845731       1 qemu.go:257] 0.00
I0307 12:03:40.749676       1 qemu.go:257] 1.00
I0307 12:03:41.689437       1 qemu.go:257] 2.00
I0307 12:03:42.355167       1 qemu.go:257] 3.01
I0307 12:03:43.066549       1 qemu.go:257] 4.01
I0307 12:03:43.825240       1 qemu.go:257] 5.01
I0307 12:03:44.813843       1 qemu.go:257] 6.01
I0307 12:03:45.889127       1 qemu.go:257] 7.02
I0307 12:03:46.994402       1 qemu.go:257] 8.02
I0307 12:03:48.342261       1 qemu.go:257] 9.02
I0307 12:03:49.296064       1 qemu.go:257] 10.02
I0307 12:03:50.639658       1 qemu.go:257] 11.02
I0307 12:03:52.954233       1 qemu.go:257] 12.03
I0307 12:03:53.813618       1 qemu.go:257] 13.03
I0307 12:03:54.765164       1 qemu.go:257] 14.03
I0307 12:03:56.615370       1 qemu.go:257] 15.03
I0307 12:03:57.603956       1 qemu.go:257] 16.04
I0307 12:03:58.769894       1 qemu.go:257] 17.04
I0307 12:03:59.819595       1 qemu.go:257] 18.04
I0307 12:04:01.927565       1 qemu.go:257] 19.04
I0307 12:04:03.161597       1 qemu.go:257] 20.04
I0307 12:04:04.555816       1 qemu.go:257] 21.05
I0307 12:04:05.943461       1 qemu.go:257] 22.05
I0307 12:04:08.140528       1 qemu.go:257] 23.05
I0307 12:04:09.424255       1 qemu.go:257] 24.05
I0307 12:04:10.330730       1 qemu.go:257] 25.06
I0307 12:04:11.062021       1 qemu.go:257] 26.06
I0307 12:04:11.973419       1 qemu.go:257] 27.06
I0307 12:04:18.593630       1 qemu.go:257] 28.06
I0307 12:04:22.055427       1 qemu.go:257] 29.06
I0307 12:04:24.884722       1 qemu.go:257] 30.07
I0307 12:04:26.791225       1 qemu.go:257] 31.07
I0307 12:04:29.608393       1 qemu.go:257] 32.07
I0307 12:04:32.603661       1 qemu.go:257] 33.07
I0307 12:04:33.606206       1 qemu.go:257] 34.08
I0307 12:04:34.829948       1 qemu.go:257] 35.08
I0307 12:04:35.672446       1 qemu.go:257] 36.08
I0307 12:04:36.477719       1 qemu.go:257] 37.08
I0307 12:04:37.401603       1 qemu.go:257] 38.08
I0307 12:04:38.232870       1 qemu.go:257] 39.09
I0307 12:04:39.040414       1 qemu.go:257] 40.09
I0307 12:04:39.807975       1 qemu.go:257] 41.09
I0307 12:04:40.608658       1 qemu.go:257] 42.09
I0307 12:04:41.319462       1 qemu.go:257] 43.10
I0307 12:04:42.077792       1 qemu.go:257] 44.10
I0307 12:04:42.819234       1 qemu.go:257] 45.10
I0307 12:04:43.702368       1 qemu.go:257] 46.10
I0307 12:04:44.640885       1 qemu.go:257] 47.10
I0307 12:04:45.450343       1 qemu.go:257] 48.11
I0307 12:04:46.220950       1 qemu.go:257] 49.11
I0307 12:04:47.076633       1 qemu.go:257] 50.11
I0307 12:04:47.909870       1 qemu.go:257] 51.11
I0307 12:04:48.681887       1 qemu.go:257] 52.11
I0307 12:04:49.460011       1 qemu.go:257] 53.12
I0307 12:04:50.352281       1 qemu.go:257] 54.12
I0307 12:04:51.128488       1 qemu.go:257] 55.12
I0307 12:04:51.981249       1 qemu.go:257] 56.12
I0307 12:04:52.832416       1 qemu.go:257] 57.13
I0307 12:04:53.677531       1 qemu.go:257] 58.13
I0307 12:04:54.496943       1 qemu.go:257] 59.13
I0307 12:04:55.427316       1 qemu.go:257] 60.13
I0307 12:04:56.851398       1 qemu.go:257] 61.13
I0307 12:04:58.062874       1 qemu.go:257] 62.14
I0307 12:04:59.464296       1 qemu.go:257] 63.14
I0307 12:05:00.673475       1 qemu.go:257] 64.14
I0307 12:05:01.784390       1 qemu.go:257] 65.14
I0307 12:05:02.719553       1 qemu.go:257] 66.18
I0307 12:05:04.410326       1 qemu.go:257] 67.18
I0307 12:05:05.521215       1 qemu.go:257] 68.18
I0307 12:05:06.284509       1 qemu.go:257] 69.19
I0307 12:05:07.110314       1 qemu.go:257] 70.19
I0307 12:05:07.983852       1 qemu.go:257] 71.19
I0307 12:05:08.927819       1 qemu.go:257] 72.19
I0307 12:05:10.465950       1 qemu.go:257] 73.19
I0307 12:05:11.250956       1 qemu.go:257] 74.20
I0307 12:05:12.000834       1 qemu.go:257] 75.20
I0307 12:05:12.722558       1 qemu.go:257] 76.20
I0307 12:05:13.636743       1 qemu.go:257] 77.20
I0307 12:05:15.093486       1 qemu.go:257] 78.21
I0307 12:05:16.071400       1 qemu.go:257] 79.21
I0307 12:05:17.021449       1 qemu.go:257] 80.21
I0307 12:05:17.796105       1 qemu.go:257] 81.21
I0307 12:05:18.774796       1 qemu.go:257] 82.21
I0307 12:05:19.422665       1 qemu.go:257] 83.22
I0307 12:05:22.673942       1 qemu.go:257] 84.22
I0307 12:05:22.805443       1 qemu.go:257] 85.24
I0307 12:05:23.524316       1 qemu.go:257] 86.24
I0307 12:05:24.464982       1 qemu.go:257] 87.24
I0307 12:05:24.882805       1 qemu.go:257] 88.25
I0307 12:05:25.080163       1 qemu.go:257] 89.35
I0307 12:05:26.106777       1 qemu.go:257] 90.37
I0307 12:05:26.499847       1 qemu.go:257] 91.39
I0307 12:05:28.600971       1 qemu.go:257] 92.39
I0307 12:05:29.471800       1 qemu.go:257] 93.39
I0307 12:05:29.905051       1 qemu.go:257] 94.40
I0307 12:05:30.399334       1 qemu.go:257] 95.45
I0307 12:05:30.949135       1 qemu.go:257] 96.49
I0307 12:05:31.128077       1 qemu.go:257] 97.53
I0307 12:05:32.400997       1 qemu.go:257] 98.53
I0307 12:05:34.830050       1 qemu.go:257] 99.53
I0307 12:06:12.094384       1 data-processor.go:282] New phase: Resize
W0307 12:06:12.251009       1 data-processor.go:361] Available space less than requested size, resizing image to available space 31642877952.
I0307 12:06:12.251124       1 data-processor.go:372] Expanding image size to: 31642877952
I0307 12:06:12.831198       1 data-processor.go:288] Validating image
I0307 12:06:12.855083       1 data-processor.go:282] New phase: Complete
I0307 12:06:12.855677       1 importer.go:189] Import Complete
```

仮想マシン(VirtualMachineリソース)のマニフェストを用意。

```bash
$ cat focal-ubuntu.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  creationTimestamp: 2022-03-07T00:00:00Z
  generation: 1
  labels:
    kubevirt.io/os: linux
  name: focal-ubuntu
spec:
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        kubevirt.io/domain: focal-ubuntu
    spec:
      domain:
        cpu:
          cores: 2
        devices:
          disks:
          - disk:
              bus: virtio
            name: disk0
          - cdrom:
              bus: scsi
              readonly: true
            name: cloudinitdisk
        resources:
          requests:
            memory: 4096M
      volumes:
      - name: disk0
        persistentVolumeClaim:
          claimName: focal-ubuntu
      - cloudInitNoCloud:
          userData: |
            #cloud-config
            hostname: ubuntu
            timezone: Asia/Tokyo
            ssh_pwauth: True
            password: ubuntu
            chpasswd: { expire: False }
            disable_root: false
            ssh_authorized_keys:
            - ssh-rsa YOUR_SSH_PUB_KEY_HERE
        name: cloudinitdisk
```

VirtualMachineリソースを作成するとPodが作成され、VMが起動してくる。

```bash
$ kubectl get pod
NAME                               READY   STATUS    RESTARTS   AGE
virt-launcher-focal-ubuntu-tb8fj   1/1     Running   0          8m6s
$ kubectl get vm  
NAME           AGE     STATUS    READY
focal-ubuntu   8m15s   Running   True
$ kubectl get vmi
NAME           AGE     PHASE     IP              NODENAME               READY
focal-ubuntu   8m17s   Running   10.244.45.93    node-1.k8s.home.arpa   True
```

Serviceリーソスを作成するためにlablesをメモ。

```bash
$ kubectl describe pod virt-launcher-focal-ubuntu-tb8fj  | head -n 9
Name:         virt-launcher-focal-ubuntu-tb8fj
Namespace:    default
Priority:     0
Node:         node-1.k8s.home.arpa/192.168.114.14
Start Time:   Mon, 07 Mar 2022 21:11:06 +0900
Labels:       kubevirt.io=virt-launcher
              kubevirt.io/created-by=75ced691-8fd0-4bee-9b86-468578bb0b88
              kubevirt.io/domain=focal-ubuntu
              vm.kubevirt.io/name=focal-ubuntu
```

Serviceリソースのセレクタに上記でメモしたlabelsを記載。

```bash
apiVersion: v1
kind: Service
metadata:
  name: focal-ubuntu-svc-lb
  labels:
    app.kubernetes.io/name: focal-ubuntu-svc-lb
  annotations:
    external-dns.alpha.kubernetes.io/hostname: focal-ubuntu.k8s.arpa
spec:
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 22
    targetPort: 22
  selector:
    kubevirt.io: virt-launcher
    kubevirt.io/created-by: 75ced691-8fd0-4bee-9b86-468578bb0b88
    kubevirt.io/domain: focal-ubuntu
    vm.kubevirt.io/name: focal-ubuntu
```

作成したVirtualMachineリソースにSSH接続できた。

![VirtualMachine](/images/running-kubevirt-on-raspi-kubernetes/VirtualMachine.png)


&nbsp;



# 最後に

VMを再起動するとIPが付与されなくなるのどうにかしたい(何か設定があるのだろう)。

細かい設定がわからないので勉強していきたい。

KubeVirt、キャッチアップしていきたい。

