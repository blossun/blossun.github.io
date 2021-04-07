---
title: "[컨트롤러] Replication Controller, ReplicaSet"
excerpt: "Auto Healing, Auto Scaling, Software Update, Job"
toc: true
toc_sticky: true

categories:
  - INFRA/kubernetes
tags:
  - KUBERNETES
  - LECTURE
---

*본문은 인프런의 [**대세는 쿠버네티스 [초급~중급]**](https://www.inflearn.com/course/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EA%B8%B0%EC%B4%88#) 강의를 정리한 글입니다.*

*모든 본문의 내용과 이미지의 출처는 강의자료에 있습니다.*

---
# Controller

* 서비스를 관리하고 운영하는데 도와주는 기능을 제공
* 컨트롤러를 삭제하면 연결된 pod들도 삭제된다. (주의!!)
* pod들은 삭제되지 않고 컨트롤러만 삭제할 수 있는 옵션이 있다.
  *  `--cascade=false`
  * 대시보드에서 사용할 수 없고, 직접 master 에서 kubectl 명령을 날려서 삭제해야 함



![image-20210407214505461](/assets/images/INFRA/kubernetes/image-20210407214505461.png)

## Auto Healing

* Node 위에 Pod이 있는데 이 Pod가 갑자기 다운되거나 Pod가 스케쥴링 되어있는 Node가 다운되면 이 Pod에서 돌아가던 서비스에 장애가 발생한다.
* 컨트롤러는 이를 즉각적으로 인지하고 Pod를 새로운 Node에 만들어준다.

## Auto Scaling

* Pod의  리소스가 Limit 상태가 되었을 때, 컨트롤러는 이 상태를 파악하고 Pod를 하나 더 만들어 주는 것으로 부하를 분산시키고, Pod가 죽지 않도록 해준다. 
* 성능에 대한 장애없이 안정적인 서비스를 제공할 수 있다.

## Software Update

* 여러 Pod에 대한 버전을 업그레이드 해야하는 경우, 컨트롤러를 통해 한번에 쉽게할 수 있고, 업그레이드 중에 문제가 생기면 롤백을 할 수 있다.

## Job

* 일시적인 작업을 해야하는 경우, 컨트롤러가 필요한 순간에만 Pod를 만들어서 해당 작업을 이행하고 삭제한다.
* 이렇게 하면 그 순간에만 자원이 사용되고, 작업 후에 다시 반환되기 때문에 효율적인 자원활용이 가능해진다.



쿠버네티스의 여러 오브젝트들이 이런 컨트롤러의 역할을 지원해준다. 그 중 몇가지를 살펴보자



# Replication Controller, ReplicaSet 

![image-20210407214808395](/assets/images/INFRA/kubernetes/image-20210407214808395.png)

Replication Controller

* 현재 Deprecated된 오브젝트
* Template, Replicas 기능을 가지고 있다.
* Selector 기능은 있지만 ReplicaSet의 기능과 차이가 있음
* (현재 아직 많이 사용중)



ReplicaSet

* Replication Controller 대체로 나온 오브젝트

* Template, Replicas 기능을 가지고 있다.
* Selector에 확장된 기능을 가짐

## Template

* 이 컨트롤러와 Pod는 Service - Pod 처럼 Label과 Selector로 연결된다.

  * Label이 붙어있는 Pod이 있고, 여기에 Selector로 매핑되는 컨트롤러를 만들면 연결된다.

* 컨트롤러를 만들 때, template으로 Pod의 내용을 넣게 된다.

  * 컨트롤러는 Pod가 죽으면 재생성 하는데, 여기 template에 있는 Pod의 정보로 Pod를 새로 만든다.

* 이러한 특성을 사용해서 App에 대한 업그레이드를 할 수 있다.

  1. template에 v2에 대한 Pod를 업데이트 한다.

  2. 기존에 있는 pod를 다운 시킨다.
  3. 컨트롤러는 템플릿을 가지고 다시 pod를 재생성하려고 하기 때문에, 새로 업그레이드된 버전의 Pod가 만들어지면서 버전 업그레이드를 수동으로 할 수 있다.

* 컨트롤러의 템플릿에도 `template: label: type: web` 으로 라벨을 붙여줘야 이 컨트롤러와 연동된다.

* template에 pod name이 중복되면 안되는데 재생성될 때 pod 네이밍이 어떻게 될까?

  → pod 네이밍이 무시되고 새로운 이름으로 만들어 준다.

  ![image-20210407221338688](/assets/images/INFRA/kubernetes/image-20210407221338688.png)

## Replicas

* replicas의 수 만큼 Pod의 갯수가 관리된다.

  ex) replicas가 1이면 Pod가 삭제되면 하나의 Pod만 재생성 해준다.

* replicas 수를 3으로 늘리면

  * 그 수만큼 pod가 늘어나면서 Scale Out이 된다.

  * 관리되는 pod 3개가 모두 지워지면 다시 3개를 재생성해준다.

* replicas 수를 줄이면 Scale In이 된다.

* template 기능과 replicas 기능을 합해서 pod와 컨트롤러를 따로 만들지 않고, 한번에 만들 수 있다.

  ex) replicas : 2와 templete 에 Pod내용을 담아서 Pod 없이 컨트롤러만 만들면,

  컨트롤러는 replicas가 2인데 현재 연결되어있는 Pod이 없기 때문에, templete에 있는 pod 정보를 가지고 2개의 pod를 생성한다.

## Seletor

Replication Selector

* key와 value값이 같은 pod들과 연결해준다.
* key와 value 중 하나라도 값이 다르면 연결하지 않는다.

ReplicaSet

* 2가지 추가적인 속성이 있다.

* `matchLabels` : Replication 컨트롤러와 같이 key, value 값이 모두 똑같아야 연결해주는 기능

  * yaml 설정에 key와 value값이 들어간다.

* `matchExpressions` : key, value를 좀 더 디테일하게 컨트롤 할 수 있다.

  ex) key: ver, operator: Exists 라고 넣으면 value는 다르지만 Label의 key가 `ver`인 모든 pod들을 선택하게 된다.

  * yaml 설정에 key와 operator 속성 값이 들어값다.

  * operator 속성값 4가지 종류

    * Exitsts : 내가 key를 정하고 그  key에 맞는 Pod들을 연결

    * DoesNotExist : 내가 key를 정하고, key에 내가 정한 key가 들어가지 않은 pod들을 연결

    * In : key, values를 설정할 수 있다. 

      ex) key : A, Values : 2, 3 → key가 A이고 Value가 2 또는 3인 Pod들을 선택

    * NotIn : key, values를 설정할 수 있다. 

      ex) key : A, Values : 2, 3 → key가 A이고 Value가 2 또는 3이 아닌 Pod들을 선택

* 실습

  * 주의

     `selector: matchLabels:` 의 내용이 `template: metadata: labels:` 의 내용과 일치해야 만들어진다.

  ```yaml
  apiVersion: apps/v1
  kind: ReplicaSet
  metadata:
    name: replica1
  spec:
    replicas: 1
    selector:
      matchLabels:
        type: web
        ver: v1
      matchExpressions:
      - {key: type, operator: In, values: [web]}
      - {key: ver, operator: Exists}
    template:
      metadata:
        labels:
          type: web
          ver: v1
          location: dev
      spec:
        containers:
        - name: container
          image: kubetm/app:v1
        terminationGracePeriodSeconds: 0
  ```

  



## Updating Controller : ReplicationController -> ReplicaSet

### ReplicationController

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: replication1
spec:
  replicas: 2
  selector:
    cascade: "false"
  template:
    metadata:
      labels:
        cascade: "false"
    spec:
      containers:
      - name: container
        image: kubetm/app:v1
```

### Kubectl

* 기존 컨트롤러에 연결된 pod은 삭제되지 않도록 하고, 기존 컨트롤러만 삭제

```sh
kubectl delete replicationcontrollers replication1 --cascade=false
```

### ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replica2
spec:
  replicas: 2
  selector:
    matchLabels:
      cascade: "false"
  template:
    metadata:
      labels:
        cascade: "false"
    spec:
      containers:
      - name: container
        image: kubetm/app:v1
```
