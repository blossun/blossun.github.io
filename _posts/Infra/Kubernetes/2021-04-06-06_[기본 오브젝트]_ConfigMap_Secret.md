---
title: "[기본 오브젝트] ConfigMap, Secret"
excerpt: "ConfigMap, Secret 오브젝트를 사용해야하는 환경과 방법"
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
# ConfigMap, Secret

![image-20210405001613733](/assets/images/INFRA/kubernetes/image-20210405001613733.png)

## ConfigMap, Secret 오브젝트를 사용해야하는 환경

* Dev(개발) 환경과 Production(상용) 환경이 있다.

* `A Service` 가 있는데 일반 접근과 보안 접근을 지원한다.

  * DEV 환경
    * `SSH : False` : 그래서 개발환경에서는 이 보안 접근을 해제할 수 있는 옵션이 있다.
    * 보안 접근을 한다면 `User` 와 `Key`를 셋팅할 수 있다.

* 이렇게 설정하고 쓰는데, 상용 환경으로 배포한다면 이 값이 바뀌어야 한다.

  *  Production 환경
    * `SSH : True` : 보안 접속으로 설정
    * `User` 와 `Key` 값도 변경해야 한다.

* 이 값들은 Container 안의 Service 이미지에 들어있는 값이기 때문에 이 값을 바꾼다는 것은

  "개발환경과 상용환경의 Container 이미지를 각각 관리하겠다"는 의미이다.

  * 이 값 몇개 때문에 큰 용량의 이미지를 별도로 관리한다는 것은 부담되는 일이다.

* 보통 이렇게 환경에 따라 변하는 값들은 **외부에서 결정**할 수 있게 한다.

* 이를 도와주는 것이 **ConfigMap**과 **Secret** 오브젝트 이다.

* **ConfigMap** : 내가 분리해야하는 일반적인 상수들을 모음

* **Secret** : Key와 같이 보안적인 관리가 필요한 값

* 그리고 Pod 생성 시 이 두 오브젝트를 연결할 수 있다.

* 연결하면, Container의 환경변수에 이 데이터들이 들어가게 된다.

* 그리고 A Service 입장에서는 이 환경변수의 값을 읽어서 로직을 처리하게 되면 위와 똑같은 역할을 할 수 있다.



* 다음과 같은 이미지를 하나 만들어 두면 상용 및 개발환경에서 사용할 수 있다.

  ![image-20210405002638745](/assets/images/INFRA/kubernetes/image-20210405002638745.png)

* 상용 환경에서는 ConfigMap과 Secret에 데이터만 변경해주면 똑같은 Container 이미지를 사용해서 원하는 기능을 사용할 수 있게 된다.



## 사용 방법

![image-20210405001623981](/assets/images/INFRA/kubernetes/image-20210405001623981.png)

* ConfigMap과 Secret을 만들 때, 데이터로 `Literal(상수)` , `File`을 넣을 수 있다.
* 그리고 `File` 로 넣을 때는 환경변수(ENV)가 아닌 Volume을 마운트해서 사용할 수 있다.



### 1. 상수를 환경변수에 넣는 방법

* ConfigMap은 Key와 Value로 구성되어 있다.

* 필요한 상수를 정의해놓으면 Pod를 생성할 때, 이 ConfigMap을 가져와서 Container 안에 환경변수로 셋팅할 수 있다.

* Secret은 Value을 넣을 때, **Base64 Encoding을 해서 넣어야한다.** (보안적 요소는 아니고, Value 규칙이다.)

* Secret의  Value값이 pod로 주입될 때는 자동으로 Decoding돼서 환경변수에서는 원래의 값이 보인다.

* Secret의 보안적 요소

  * 일반적인 오브젝트 값들은 쿠버네티스의 DB에 값이 저장되는데, Secret은 `Memory에 저장`이 된다.

  * 파일에 저장되는 것보다 메모리 저장이 보안상 더 낫다.

  * ConfigMap은 Key, Value값을 무한히 넣을 수 있는 반면에, Secret은 `1Mbyte` 용량 만큼만 넣을 수 있다.

    하지만 이 Secret은 메모리에 저장된다고 했으므로, 너무 많이 만들게 되면 시스템 자원에 영향을 끼치므로 주의가 필요

* 이미지 예시

  ### 1-1) ConfigMap

  * 주의 ) Key, Value는 모두 String 값이기 때문에 Boolean 값을 넣으려면 쿼터(`'` )로 감싸서 String 값으로 인식하도록 만들어줘야 한다.

    안그러면 에러 발생

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: cm-dev
  data:
    SSH: 'false'
    User: dev
  ```

  ### 1-2) Secret

  * 주의 ) `data: Key:` 의 값은 `Base64`로 인코딩한 문자값을 넣어줘야 한다.

    안그러면 Base64 인코딩하라고 에러 발생

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: sec-dev
  data:
    Key: MTIzNA==
  ```

  * 대시보드에서 내용을 볼 수 있는 기능이 있어서 Key의 평문값을 볼 수 있다.

    * **실질적으로 대시보드를 운영에서 사용하지 않는 이유가 이런 중요한 값들이 노출되기 때문에 보안상의 문제로 사용하지 않는다.**

    ![image-20210405010557962](/assets/images/INFRA/kubernetes/image-20210405010557962.png)

  ### 1-3) Pod

  * `spec: containers: envFrom: configMapRef:` : ConfigMap을 레퍼런스

    `name` : 가져올 ConfigMap의 이름

  * `spec: containers: envFrom: secretRef:` : Secret을 레퍼런스

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-1
  spec:
    containers:
    - name: container
      image: kubetm/init
      envFrom:
      - configMapRef:
          name: cm-dev
      - secretRef:
          name: sec-dev
  ```

  

### 2. File을 환경변수에 넣는 방법

* File을 통으로 ConfigMap에 담을 수 있다.
* 이 때, `File 이름`이 `Key`가 되고, `File 내용`이 `Value`가 돼서 ConfigMap이 만들어 진다.
* 이  ConfigMap을 환경변수에 넣을 때, 파일 이름을 Key로 하면 이상하니 Key를 새로 정의해서 Content만 넣는다.

#### 2-1) Configmap

* File을 ConfigMap으로 만드는 것은 대시보드에서 지원해주지 않는다.
* 직접 Master 콘솔로 접속해서 `kubectl` 명령을 실행한다.
* "`cm-file` 이라는 configmap을 만들고, `file-c.txt` 파일을 넣을 것이다"라는 명령

```sh
echo "Content" >> file-c.txt
kubectl create configmap cm-file --from-file=./file-c.txt
```

#### 2-2) Secret

* 주의! **이 명령을 통해 파일 내용이  Base64로 인코딩된다.**
* 파일 내용이 이미 Base64로 인코딩 되어있는 상태라면 두 번 인코딩 되는 것이므로 이를 유의할 것!!!

```sh
echo "Content" >> file-s.txt
kubectl create secret generic sec-file --from-file=./file-s.txt
```

#### 2-3) Pod

* `spec: containers: env: name: file-c` : 컨테이너에 `file-c`라는 이름의 환경변수를 넣을 것이다.
* `valueFrom:` : 환경 변수의 값을 가져온다.
  * `configMapKeyRef: name: cm-file key: file-c.txt` : ConfigMap의 Key를 레퍼런스 한다. ConfigMap의 이름은 `cm-file`이고, cm-file 안에 있는 `file-c.txt`라는 `Key`에 대한 `Value`를 넣게 된다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-file
spec:
  containers:
  - name: container
    image: kubetm/init
    env:
    - name: file-c
      valueFrom:
        configMapKeyRef:
          name: cm-file
          key: file-c.txt
    - name: file-s
      valueFrom:
        secretKeyRef:
          name: sec-file
          key: file-s.txt
```

<img src="/assets/images/INFRA/kubernetes/Screen%20Shot%202021-04-05%20at%201.10.03%20AM.png" alt="Screen Shot 2021-04-05 at 1.10.03 AM" style="zoom:33%;" />



### 3. File을 마운팅하는 방법

* File을 ConfigMap에 담는 것은 2번과 동일하다
* Pod를 만들 때, Container 안에 Mount Path를 정의(`/mount`)하고, 이 Path안에 파일을 마운팅할 수 있다.

------

![ConfigMap, Secret with Mount for Kubernetes](/assets/images/INFRA/kubernetes/ConfigMap,%20Secret%20with%20Mount%20for%20Kubernetes.jpg)

### 3-1) Pod

* `spec: containers: volumeMounts:` : 컨테이너 안에 volume을 마운트
    - `name: file-volume` : 마운트할 volume 이름
      `mountPath: /mount` : moutPath
  * ` volumes:` : 볼륨정보
      - name: file-volume
      - `configMap: name: cm-file` : 이 volume 안에 configMap을 담는다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-mount
spec:
  containers:
  - name: container
    image: kubetm/init
    volumeMounts:
    - name: file-volume
      mountPath: /mount
  volumes:
  - name: file-volume
    configMap:
      name: cm-file
```

마운트한 경로에 configMap으로 정의한 파일이 들어가 있다.

<img src="/assets/images/INFRA/kubernetes/image-20210405011224749.png" alt="image-20210405011224749" style="zoom: 50%;" />

![image-20210405011339407](/assets/images/INFRA/kubernetes/image-20210405011339407.png)



### File in Env  vs Volume Mount

* pod 생성 후, ConfigMap의 내용을 변경한다면?
  * ENV (환경변수) 방식
    * 한 번 주입하면 끝
    * ConfigMap의 데이터가 변해도 Pod의 환경변수 값에 영향이 없다.
    * Pod가 죽어서 재생성이 돼야지만 변경된 값을 다시 받아와서 수정할 수 있다.
  * Volume Mount 방식
    * Mount는 원본(여기서 ConfigMap)과의 연결이므로
    * ConfigMap의 내용이 변경되면 Pod에 마운팅된 내용도 변경된다.
* 이러한 특성을 알고 필요한 상황에 따라서 적절히 활용하면 된다.

