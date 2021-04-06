---
title: "[기본 오브젝트] Namespace, ResourceQuota, LimitRange 오브젝트를 써야하는 이유"
excerpt: "발생할 수 있는 문제"
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
# Namespace, ResourceQuota, LimitRange 오브젝트를 써야하는 이유

![image-20210405011622144](/assets/images/INFRA/kubernetes/image-20210405011622144.png)

* Kubernetes Cluster : 전체 사용할 수 있는 자원이 있다.
* 일반적으로 Memory, CPU 등이 있다.
* Cluster 내에는 여러 Namespace를 만들 수 있고, Namespace 내에는 여러 Pod들을 만들 수 있다.
* 각 Pod는 필요한 자원을 Cluster 자원을 공유해서 사용한다.

## 발생할 수 있는 문제 1

* 만약 한 Namespace 안에 있는 pod가 Cluster에 남는 자원을 모두 사용하면, 다른 pod 입장에선 더 이상 쓸 자원이 없어서 자원이 필요할 때 문제가 발생한다.

### 해결방법

* 이러한 문제를 해결하기 위해서 `ResourceQuota(리소스쿼터)`가 존재한다.
  * 이를 Namespace마다 달게되면, **<u>각 Namespace마다 최대 한계를 설정할 수 있다.</u>**
  * 따라서 하드 자원이 이 한계를 넘을 수 없고, 3번째 Pod 입장에서 자원이 부족해서 문제가 될지언정 다른 Namespace에 있는 Pod에는 영향을 끼치지 않게 해준다.

## 발생할 수 있는 문제 2

* 한 Pod가 자원 사용량을 너무 크게 해버리면 다른 Pod들이 이 Namespace에 더이상 들어올 수 없게 된다.

### 해결방법

* 이러지 못하게 관리하기 위해서 `LimitRange`를 둬서 **<u>Namespace에 들어오는 pod의 크기를 제한할 수 있다.</u>**

  * 한 Pod의 자원 사용량이 `LimitRange`에 설정된 값보다 낮아야 Namespace에 들어올 수 있다.

  * 클 경우 들어올 수 없다.



* ResourceQuota, LimitRange 오브젝트는 Namespace 뿐만 아니라 **Cluster에도 달아서 전체 자원에 대한 제한을 걸 수 있다.**



# 오브젝트 기능 설명

![image-20210405012854498](/assets/images/INFRA/kubernetes/image-20210405012854498.png)

## Namespace

* 한 Namespace 내에는 같은 타입의 오브젝트는 이름이 중복될 수 없다.
  * 같은 Pod의 이름을 중복해서 만들 수 없다.
  * 오브젝트들마다 별도의 uuid가 존재하지만 한 Namespace 안에서는 같은 종류의 오브젝트라면 이름 또한 uuid 값 같이 유일한 키 역할을 할 수 있게된다.
* 타 Namespace와 분리돼서 관리된다.
  * 이미지 예시
    * Namespace2에 Service1가 존재한다. Pod와 Service간의 연결을 Pod에는 Label을 달고 Service에는 Selector를 달아서 연결하지만, **타 Namespace에 있는 Pod과 Service는 연결할 수 없다.**
  * 대부분의 자원들은 그 자원을 만든 Namespace 안에서만 사용할 수 있다.
  * Node나 PV와 같이 모든 Namespace에서 공용으로 사용되는 오브젝트도 있다.
    * NodePort는 Namespace 별로 나눌 수 없는 부분임
      * 예시 ) Namespace1에 NodePort type의 Service를 30000번 포트로 만들면 Namespace2에 동일한 30000번 포트로 NodePort type의 Service를 만들 수 없다.
* Namespace를 지우게 되면 Namespace 내의 자원들도 모두 지워진다. (Namespace를 지울 때 이를 유의할 것!!!)

* 설정파일
  * Namespace : 이름 외에 별도 설정이 없다.
  * Pod과 Service 의 `namespace` : 내가 속할 Namespace를 지정

#### Namespace로 분리되지 않는 기능

* Pod 마다 IP를 가지고 있다. 그렇다면 타 Namespace에서 Pod의 IP로 접근하려고 한다면?
  * 기본적으로는 연결이 된다.
  * (추후에 배울) `Network Policy`를 통해서 제한할 수 있다.
* 다른 Namespace의 Pod에서 사용하는 host-path volume을 타 Namespace의  Pod에서 mountPath의 내용을 볼 수 있다?
  * Pod의 권한이 기본적으로 Root로 사용하기 때문이다.
  * 추후 `Pod Security Policy`에서 User 권한을 주도록 변경이 필요하다.



## ResourceQuota

* Namespace의 자원한계를 설정하는 오브젝트

* Namespace의 제한하고 싶은 자원을 명시해서 ResourceQuota를 Namespace에 달아준다.

* 설정 예시

  <img src="/assets/images/INFRA/kubernetes/image-20210405014107534.png" alt="image-20210405014107534" style="zoom:33%;" />

  * Namespace에 들어갈 Pod들의 전체 request 자원들을 최대 3Gi로 설정하겠다.
  * Memory의 Limit은 6Gi로 하겠다.

* **ResourceQuota가 지정된 Namespace에 Pod를 만들 때, Pod는 무조건 해당하는 스팩(여기선 requests, limits)을 명시해야 한다.**

* 스팩이 없으면 이 Namespace에 Pod가 만들어지지 않는다.

  <img src="/assets/images/INFRA/kubernetes/image-20210405093546312.png" alt="image-20210405093546312" style="zoom: 33%;" />

* 현재 Pod1이 Namespace에 만들어진 상태라면 허용된 3Gi 중 2Gi의 메모리를 사용중이기 때문에 1Gi 밖에 남지 않아서 Pod3이 사용할 자원양이 부족하기 때문에 만들어지지 않는다.

  ![image-20210405094119700](/assets/images/INFRA/kubernetes/image-20210405094119700.png)

* 제한 가능한 항목

  * Compute Resource : cpu, memory, storage ...
  * Object Count : Pod, Service, ConfigMap, PVC ... 생성가능한 오브젝트 수를 제한
  * 모든 Object를 제한할 수 있는 것은 아니다. 쿠버네티스 버전이 업데이트 되면서 제한할 수 있는 Object가 늘어나고 있다.
  * 따라서 지금 버전에서 어느 Object 까지 제한할 수 있는지 확인하고 사용하여야 한다.

* 설정파일

  * ResourceQuota
    * namespace : 할당할 Namespace 지정
    * `spec: hard: ` : 제한할 종류, 자원, 한계치

### 주의사항

* **ResourceQuota를 만들기 전에 해당 Namespace에 이미 만들어진 Pod가 존재하지 않도록 해야한다.**
* Pod가 존재하는 Namespace에 추가적으로 ResourceQuota를 달아주려고 하면?
  * ResourceQuota에 명시된 양보다 더 많은 양의 자원을 사용하는 상황이 발생한다.
  * ResourceQuota에 memory를 1Gi로 제한했는데 이미 존재하는 pod들이 총 1Gi를 사용 중이라 하더라도 pod가 새롭게 만들어진다.
  * ResourceQuota가 달아진 시점 이후로 만들어지는 오브젝트들만 제한된다.



* 실습

  ### 2-2) ResourceQuota

  ```yaml
  apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: rq-1
    namespace: nm-3
  spec:
    hard:
      requests.memory: 1Gi
      limits.memory: 1Gi
  ```

  ResourceQuota Check Command (대시보드에서 확인할 수 없고 콘솔로 확인해야 함)

  ```sh
  kubectl describe resourcequotas --namespace=nm-3
  ```

  ![image-20210405093500696](/assets/images/INFRA/kubernetes/image-20210405093500696.png)



## LimitRange

* 각각의 Pod 마다 Namespace에 들어올 수 있는지 자원을 체크해준다.
* 체크되는 항목
  * `min`
    * `memory: 1Gi` :  pod에서 설정되는 Memory의 limit 값이 1Gi가 넘어야 한다.
  * `max`
    * `memory: 4Gi` : pod에서 설정되는 Memory의 limit 값이 4Gi를 초과할 수 없다.
  * `maxLimitRequestRatio`
    * `memory: 3` : Request 값과 Limit의 값의 비율이 최대 3배를 넘으면 안된다.
    * 예시의 Pod2 - 비율이 4배이므로 들어올 수 없다.

* 옵션
  * `defaultRequest` , `default` : Pod에 아무런 스팩 설정을 하지 않았을 때 Pod에 자동으로 명시된 값이 설정된다.
* **ResourceQuota는 Namespace 뿐만 아니라 Cluster 전체에 부여할 수 있는 권한이지만,**
  **LimitRange의 경우 Namespace내에서만 사용 가능합니다.**

* 설정파일
  * LimitRange
    * namespace : 할당할 Namespace 지정
  * `limit: type:` : 어떤 (제한 가능한) 항목별로 제한할지 지정
    * Container, Pod, PVC 단위 설정이 존재
    * 각 type마다 설정할 수 있는 옵션이 다르다.

### 주의사항 - LimitRange Exception

* 한 Namespace에는 하나 이상의 LimitRange가 들어갈 수 있다.

* `limits: max: memory: ` 값과 `defaultRequest: memory:` , `default: memory:` 값이 다른 두 개의 LimitRange가 있다.

  * 예상치 못한 정책으로 하드가 생성되지 않을 수 있다. (어떤 정책이 선택(?)될지 모름)

  <img src="/assets/images/INFRA/kubernetes/image-20210405103124060.png" alt="image-20210405103124060" style="zoom:33%;" />

### E-1) Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nm-6
```

### E-2) LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: lr-5
spec:
  limits:
  - type: Container
    min:
      memory: 0.1Gi
    max:
      memory: 0.5Gi
    maxLimitRequestRatio:
      memory: 1
    defaultRequest:
      memory: 0.5Gi
    default:
      memory: 0.5Gi
```

### E-3) LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: lr-3
spec:
  limits:
  - type: Container
    min:
      memory: 0.1Gi
    max:
      memory: 0.3Gi
    maxLimitRequestRatio:
      memory: 1
    defaultRequest:
      memory: 0.3Gi
    default:
      memory: 0.3Gi
```

### E-4) Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-1
spec:
  containers:
  - name: container
    image: kubetm/app
```

![image-20210405100307992](/assets/images/INFRA/kubernetes/image-20210405100307992.png)
