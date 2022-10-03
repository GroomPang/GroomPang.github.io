---
title: "Azure Container Instances Escape (KO)"
author: "GRoomPang"

catalog: true
tags:
  - [BoB, Cloud, Azure, serverless, escape, KO]

toc: true
toc_sticky: true
 
date: 2022-09-18
last_modified_at: 2022-10-03
---

## TL; DR

---

Azure Container Instances (ACI)는 Microsoft Azure에서 제공하는 Container-as-a-Service (CaaS) 서비스로, 사용자가 서버 및 인프라를 관리하지 않고 원하는 컨테이너를 동작시킬 수 있는 서비스이다. 구름빵 팀은 이러한 ACI에서 Container escape 취약점을 발견하였고, 이를 MSRC에 제보하여 취약점으로써 인정받았다.

공격자의 컨테이너를 실행하는 노드에서, 공격자는 루트 계정을 가지고 작업을 수행할 수 있다. 추가적인 연구를 통해, 쿠버네티스 클러스터 내 파드 열람, 공격자 노드의 kubelet 조작, 웹사이트에 출력되는 로그 변조 등이 가능함을 확인하였다.

## Exploration on Azure Container Instances

---

우선적으로, ACI의 컨테이너 런타임을 확인하였다. 컨테이너 이미지의 ENTRYPOINT를 `/proc/self/exe`으로 지정할 경우, 컨테이너 실행과 동시에 host의 runC 바이너리를 실행하게 된다. 아래는 그러한 컨테이너 이미지를 ACI에서 동작 시킨 뒤, Azure Portal에서 ACI 컨테이너의 출력 로그를 확인한 것이다.

![Figure 1.  container log of entrypoint /proc/self/exe at Azure Portal](https://user-images.githubusercontent.com/54650556/193565747-0567d419-b8bf-466b-ac4c-4bce82c2b108.png)
*Figure 1.  container log of entrypoint /proc/self/exe at Azure Portal*

ACI는 runC 1.0.0-rc10 버전의 컨테이너 런타임을 사용하고 있었고, 이는 ACI가 CVE-2021-30465(symlink exchange attack)에 취약하다는 것을 의미한다.

또한, ACI는 Azure Kubernetes Service (AKS)에서 동작한다. 그렇기 때문에, Kubernetes에서 pod를 생성할 때 사용할 수 있는 옵션들 중 대부분이 ACI에서도 비슷한 형태로 존재한다. 아래 두 그림을 통해, 일반적인 Kubernetes Cluster와 ACI Cluster의 차이를 확인할 수 있다.

![그림2](https://user-images.githubusercontent.com/54650556/193580492-40bd7b83-2d54-431a-ba46-adac67f73b86.png){: width="60%" height="60%"}
*Figure 2. Kubernetes Cluster*

![그림4](https://user-images.githubusercontent.com/54650556/193580498-ce2da0dc-d0c8-4f4c-bf26-3917053574d8.png){: width="60%" height="60%"}
*Figure 3. ACI Cluster*

일반적인 Kubernetes에서는 개개의 노드에서 가용한 만큼 pod를 할당하는 반면, ACI에서는 개개의 고객이 할당한 pod가 노드 1개를 차지한다. 즉, 고객 A와 고객 B가 할당한 pod가 서로 다른 노드에 할당되고, 해당 노드는 dedicated 성질을 띠고 있는 것이다.

## CVE-2021-30465 삽질

---

CVE-2021-30465는 구체적인 mount 설정을 변경할 수 있는 컨테이너를 임의로 생성하고, race condition에 의한 symlink-exchange attack을 기반으로 호스트 파일 시스템에 접근하는 취약점이다. 이 취약점은 1.0.0-rc95 이전 버전의 runC에서 트리거 가능하고, ACI pod에 emptyDir volume을 탑재하며 여러 개의 컨테이너를 이용하여 해당 취약점 트리거를 시도하고자 하였다.

ACI pod에서 해당 취약점 트리거를 시도하기에는 약간의 제약이 존재했다:

- 할당 가능한 emptyDir을 tmpfs filesystem으로 할당 불가
    
    runC의 libcontainer/rootfs_linux.go 코드를 봤을 때, tmpfs일 경우 해당 파일이 디렉토리인지 검사하는 함수와 실제 마운트를 수행하는 함수 사이의 간격이 상대적으로 길기 때문에 취약점 트리거가 수월하다. 그러나, ACI는 bind mount(ext4 filesystem)로만 emptyDir volume을 할당할 수 있었다.
    
- 이미지 PULL이 불가능한 컨테이너가 존재할 경우, pod 생성 중단
    
    로컬 쿠버네티스에서 CVE를 테스트할 때는, symlink exchange를 수행하는 컨테이너를 제외한, 다른 모든 컨테이너를 잘못된 레포지토리를 지정하여, 컨테이너들을 Pending 상태로 만들고, exchange를 수행한 뒤에 정상적인 컨테이너를 할당하는 방식으로 진행했다. 그러한 방식으로 진행할 때, 취약점을 트리거하기가 수월하였기 때문이다. 그런데, 그러한 방법을 ACI pod에서는 적용할 수 없었다.
    
- 모든 컨테이너의 시작 시간이 거의 동일함
    
    모든 컨테이너의 시작 시간이 거의 비슷하기 때문에, 직접 컨테이너로 진입해 symlink exchange를 수행하기엔 시간이 턱없이 부족할 뿐더러, 실질적인 취약점 트리거를 수행하는 컨테이너의 개수가 현저히 줄어들 것임을 의미했다.
    

로컬 환경에서 emptyDir이 ext4 파일 시스템이면서, 컨테이너가 Start와 Stop을 반복하는 방식으로 익스플로잇이 가능한 것을 확인하였다. 그러나, 중요한 것은 익스플로잇 확률이 매우 낮다는 것이었다. 로컬에서 취약점이 발생하였던 것은 CPU의 개수가 2개에 불과하여, race condition이 더욱더 잘 일어날 수 있는 환경이었기 때문으로 파악하였다.

그럼에도 불구하고 적은 수의 컨테이너를 사용했던 이유는 ACI pod의 마운트 옵션이 MS_SHARED 였기 때문이다. ACI는 AKS 위에서 돌아가지만, volume mount에 대해 [Mount Propagation](https://kubernetes.io/docs/concepts/storage/volumes/#mount-propagation)에 자동으로 `Bidirecitonal` 옵션을 추가해 마운트가 수행한다. 따라서, 여러 개의 컨테이너를 돌려도, 하나의 컨테이너가 마운트되는 순간, 해당 마운트가 symlink exchange를 수행하는 컨테이너에도 전파된다. 이미 마운트된 디렉토리를 대상으로 symlink exchange는 진행되지 않기 때문에, 모든 컨테이너가 **똑같은 마운트** 수행 결과를 가지게 된다.

![Figure2. exponential mount occuring because of bidirectional mode](https://user-images.githubusercontent.com/54650556/193566091-e1232839-e77a-4204-b684-24fec91ed942.png)*Figure 4. exponential mount occuring because of bidirectional mode*

추가로, rshared(`MS_REC` + `MS_SHARED`)로 마운트되기 때문인지, 컨테이너가 마운트를 수행할 때마다 **지수승**으로 마운트 개수가 늘어났다. 기존에 symlink exchange를 수행하는 컨테이너가 가진 마운트가 새롭게 Start하는 컨테이너로 전이되고, 그것이 다시 전파되기 때문인 것으로 예상하고 있다. 이러한 이유로 인해, 컨테이너가 마운트되는 속도가 현저히 느려졌고, 우리 팀은 적은 수의 컨테이너로 CVE-2021-30465를 계속 시도할 수밖에 없었다.

## 컨테이너 개수에 따라 달라지는 환경

---

그렇다면, “CPU 개수보다 컨테이너가 훨씬 더 많아야, 볼륨도 훨씬 더 많아야 (모든 컨테이너의 마운트 결과는 같을 수 있겠지만), **익스 확률을 더 높일 수 있겠다!**” 라는 결론으로 컨테이너마다 할당하는 CPU와 Memory의 양을 줄여서 할당하였다. ([참고](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/))

![Untitled](https://user-images.githubusercontent.com/54650556/193566623-377f1c9c-80de-4bea-8693-76c9fc89dfb8.png){: width="80%" height="80%"}

![Figure 5, 6. differing container environments](https://user-images.githubusercontent.com/54650556/193566637-8cf06374-1785-4601-8918-d3fe204b8a0b.png){: width="80%" height="80%"}
*Figure 5, 6. differing container environments*

그러자, hostname(SandboxHost-* → wk-caas-*)이 달라졌고, 추가적으로 volume의 수행 결과 및 mount 옵션도 바뀐 것을 확인할 수 있었다.

![Figure 5. the change of the container’s environment](https://user-images.githubusercontent.com/54650556/193566770-8d08d2b8-1c47-412e-8e01-f665ff9e83a2.png)
*Figure 7. the change of the container’s environment*

약간의 테스트를 수행한 결과, 5개 이상의 컨테이너를 ACI pod에 할당하거나, 구버전 쿠버네티스에서만 지원하는 gitRepo volume을 ACI pod에 할당할 때, 호스트 환경이 위와 같이 달라짐을 확인하였다. 컨테이너 런타임을 가져올 때, unit42 팀에서 개발한 이미지인 [WhoC](https://github.com/twistlock/whoc)를 사용해 로컬 환경으로 가져올 수 있었다.

![Figure 6-1. runC that we extracted](https://user-images.githubusercontent.com/54650556/193566664-c7680140-1249-4ca6-af36-7c16f8dcfdf9.png){: width="80%" height="80%"}
*Figure 8-1. 추출한 runC*

![Figure 6-2. [the vulnerable runC of Azurescape](https://www.paloaltonetworks.com/blog/2021/09/azurescape/)](https://user-images.githubusercontent.com/54650556/193566789-be69af01-e148-4f67-aac0-d40c80322603.png){: width="80%" height="80%"}
*Figure 8-2. Azuresacpe 때 취약했던 runC*

그와 동시에, 변화한 호스트의 runC 바이너리를 추출하였더니 구버전의 runC를 사용하고 있음을 확인할 수 있었고, 이는 이전에 ACI에서 발생한 취약점이었던 Azurescape에서와 동일한 버전이었다.

[Reference] Azurescape : [https://unit42.paloaltonetworks.com/azure-container-instances](https://unit42.paloaltonetworks.com/azure-container-instances/)

runC v1.0.0-rc2는 symlink exchange attack 외에도, 컨테이너 런타임 자체적으로 취약한 부분이 많기 때문에, 확률적으로 트리거되는 CVE를 사용하지 않고, 확정적으로 트리거할 수 있는 CVE를 활용하는 것이 더 낫다고 판단하였다. 그러한 취약점 중 하나가 CVE-2019-5736이다.

## Container Escape (CVE-2019-5736)

---

CVE-2019-5736은 간단히 설명하자면, 컨테이너 이미지의 ENTRYPOINT를 `/proc/self/exe`로 설정하여, 컨테이너가 만들어짐과 동시에, 호스트의 runC 바이너리를 조작하는 취약점이다.

![Figure 7. Connecting reverse shell by applying CVE-2019-5736 trigger image](https://user-images.githubusercontent.com/54650556/193566113-2e7aa838-ed02-4aa9-ad0e-fe2815adcddc.png)
*Figure 9. Connecting reverse shell by applying CVE-2019-5736 trigger image*

ACI pod에서 5개 이상의 컨테이너를 할당하거나, gitRepo volume을 사용할 때, ACI는 pod를 취약한 호스트에 배포하게 되고, 그중 하나의 컨테이너가 CVE-2019-5736을 트리거하는 이미지를 가지고 있다면, 그대로 컨테이너 이스케이프에 성공할 수 있다.

## Attempts to gain Cluster control

---

![Figure 8. the architecture of the exploited(escaped) ACI](https://user-images.githubusercontent.com/54650556/193566123-ef5b9a36-5596-4df5-b6ac-b53afe0245d2.png){: width="70%" height="70%"}
*Figure 10. the architecture of the exploited(escaped) ACI*

Container Escape를 통하여, ACI에서 제공하는 접근 범위를 넘어서 호스트까지 접근할 수 있게 되었다. 그러나, ACI에서는 개별 고객을 노드 단에서 격리하고 있기 때문에, 여전히 공격자는 다른 고객의 자원에 접근할 수 없는 상태였다. 따라서, 우리는 타 고객의 Information Disclosure 혹은 클러스터 장악으로 이어질 수 있는 취약점을 찾고자 하였다.

Docs에는 적혀 있지 않은, ACI 서비스의 운용 환경을 파악할 수는 있었으나, 추가적인 취약점은 발견할 수 없었고, 그대로 연구를 종료하였다. 아래는 공격을 수행하는 과정을 녹화한 것이다.

<iframe width="640" height="360" src="https://www.youtube.com/embed/xitHuQj8k24" title="[Azure Container Instances] Container Escape" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Azure’s Fix

---

이전에 취약한 환경이라 언급했던 서비스에서도 CVE-2019-5736에 안전하도록, runC 버전을 업데이트함으로써, ACI 서비스의 취약한 환경을 패치하였다.

### Timeline

2021년 11월 26일: Container Escape 발견

2021년 12월 09일: MSRC 취약점 제보

2022년 01월 15일: 취약점 인정

2022년 03월 18일: 취약점 패치

## Conclusion

---

ACI 서비스에서 발생하는 Container Escape 취약점은 이번이 처음이 아니다. 2021년 9월에 Paloalto의 unit42 팀에서 공개한 Azurescape 취약점이 있었다. 당시에도 ACI 서비스에서는 취약한 컨테이너 런타임을 사용하고 있었고, 동일하게 CVE-2019-5736으로 쉽게 컨테이너를 탈출할 수 있었다.

MSRC에서는 이러한 취약점을 4개 이하의 컨테이너를 배포했을 때만 안전한 컨테이너 런타임을 사용하도록 패치한 것으로 보이며, 그 결과 ACI 서비스에서 여전히 취약한 환경을 확인할 수 있었다. Azure에서는 2016년 10월에 배포한 1.0.0-rc2 runC를 사용하고 있었고, 이 버전은 컨테이너 이스케이프에 충분히 취약하다. 소프트웨어를 최신 버전으로 업데이트하는 것만으로, 보다 안전한 서비스를 구축할 수 있었을 것이다. 패치를 했는지 여부가 중요한 것이 아니라, 어떻게 패치를 했는지가 중요하다.