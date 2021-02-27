---
published: true
title: packets on k8s networks 
date: 2021-02-28
last_modified_at: 2021-02-28
categories: [youtube 로 공부]
tags: [k8s,networks]
comments: true
--- 

# 들어가며 
k8s 안에서 패킷이 어떻게 움직이는지 진짜 궁금했었습니다. 실제 어떻게 동작하는지 잘 알려주는 
훌륭하게 개념을 설명하는 유튜브를 찾았는데, 이를 공부하면서 정리를 해봤습니다. 대충 이해한거라 틀린 내용이 있을 수도 있습니다.
네트웍을 잘 몰라서 이해한대로 썼는데 언제나 그렇듯 더블체크는 늘 필요합니다.  

# Life of Packet

{{< youtube 0Omvgd7Hg1I >}}

# 정리

## network 기술 소개

* 모든 POD 은 k8s 안에서 통신가능한 진짜 IP 를 가진다. 도커와 같이 포트매핑을 하는 방식을 사용하지 않는다. 
* POD IP 는 다른 POD 에서 접근가능한 진짜 아이피다. 여기서 POD 은 같은 노드가 아닌 다른 노드에 위치할 수 있다. 
* 리눅스에서 제공하는 Network Namespace 과 virtual interface 를 통해서 이를 구현하였다.     
* network namespace 에 대한 간략한 소개. 자세한건 인터넷 찾아봐라.

기본으로 Network는 하나의 큰 공통자원이라 변경 내용이 전체 프로세스에 영향을 미칩니다. 이러한 특성은 가상화에서 큰 문제를 일으킬수 있어 
namespace 라는 개념으로 나누고, 서로 격리된 형태로 인터페이스, iptables, routing 등을 세팅할 수 있다고 합니다. 기본으로 OS 를 
설치하면 root namespace 가 있고, namespace 를 여러개 만들어서 격리를 한 후 root namespace 와 통신을 통해 외부와 통신합니다. 
다음 글에 설명이 잘 되어있습니다. ( https://www.joinc.co.kr/w/man/12/NetworkNamespace )

* PDO 별로 namespace 가 생성되고, root namespace 와 서로 연결할 수 있는 virtual interface 가 생긴다. 
* 여러개의 POD 이 생겼다면 여러개의 namespace 가 있을꺼고, root namespace 에는 이들간의 virtual interface 간의 패킷을 릴레이하기 위한 브릿지가 생성된다. 
* 브릿지도 궁금하면 검색해봐라 

검색해봤습니다. https://ops.tips/blog/using-network-namespaces-and-bridge-to-isolate-servers/ 여기 설명이 잘되어있습니다. 간단히 양 네트워크를 연결해주는 다리 라고 생각합시다.  

* 같은 노드끼리 POD 의 통신은 POD -> POD namespaced veth -> root veth -> bridge -> root veth -> POD namespaced veth -> POD 의 순으로 이뤄진다. 
* 다른 노드에 있는 POD 과 통신하기 위해서는 POD -> POD namespaced veth -> root veth -> bridge -> eth0 -> network -> eth0 -> bridge -> root veth 순으로 이루어진다. 
* Overlay 에 대한 설명. 간단히 말하면 Packet 을 또 한번 감싸는 패킷을 만들어서 사용하는 기술 ( packet-in-packet )
* Flannel 이라는 CoreOS 에서 만든 Overlay 기술을 이용한 프로그램을 사용하는 예를 보여줌. 
* POD1 에서 다른노드의 POD4 로 패킷이 전달될때 브릿지 상에 POD4 의 IP 정보는 알수없음. 그래서 flannel 이 패킷을 가로체서 앞에 해더를 덧붙임. 
* flannel 은 어느 노드에 POD4가 있는지 알고 있다. 이렇게 패킷을 한번 더 감싸 eth0 을 통해 외부로 내보내면 POD 4가 있는 노드가 받고 이 헤더를 제거 한 후 브릿지로 보냄. 
* 헤더에는 POD4 가 DST 정보로 박혀있으니 브릿지를 통해서 원하는 POD 으로 패킷이 전달됨. 

## Service 

* POD 은 언제든지 죽을 수 있고, 다시 살아나면서 다른 IP 로 바뀔수도 있고 늘어날수도 줄어들수도 있다. 그래서 POD IP 를 사용할 수 없다. 
* 그래서 서비스라는 추상화된 개념을 도입했고, 서비스는 바뀌지 않는 VIP 를 제공한다. 그 다음 POD 을 가르키도록 한다. 
* k8s 에서 POD 을 그루핑하여 가르키는 방법은 바로 selector 를 이용한 방식 
* selector 만 있다면 라우팅할때마다 k8s api 서버에 물어봐야하는데 그건 너무 비효율적. 그래서 endpoint 가 등장했다. 
* endpoint 에 pod 의 아이피를 지정해놓고 이를 가지고 라우팅. 

그래서 Service 와 Endpoint 가 한쌍으로 생기는 거였군요. 저는 이 부분이 가장 인상깊었습니다. 이제 Service 를 가지고 Packet 이 이동하는 방식을 설명해줍니다. 
 
* pod 1 이 svc 을 향해 간다면, 브릿지 이후 iptable 을 통과한다. 
* iptable 을 통과하면서 dst 는 실제 라우팅되는 pod 아이피로 변경. DNAT 이고 conntrack 이용하여 바꿈. 
* 이제 DNAT 을 통해서 eth0 을 통해 인터넷으로 나감. 이후는 위에서 설명한 개념과 동일
* 다시 돌아올때 eth0 을 통해서 패킷이 들어오면 conntrack 에 정보를 뒤져서 Source 를 바꿔줌. DNAT 한걸 풀어줌  
* 이후 는 전과 동일 
* iptable 의 설정은 kubeproxy 를 통해서 이루어짐. iptable 을 관리하는 Controller 
* 사용자들이 사용하기 쉽게 서비스는 자신의 이름을 DNS 에 등록한다. 

Service 라는 추상화의 구현체가 iptable 설정이고, VIP 라는걸 알았습니다.  
이제 외부와 통신을 어떻게 하는지 설명합니다. 

* 클러스터 외부와의 통신은 NAT 을 통해서 이루어진다. NODE 에서 8.8.8.8 을 날린다면 클러스터 레벨에서 설정된 1:1 NAT 을 통해서 source 가 Cluster 전용 IP 에서 External IP 로 변신해서 나간다. 
* 돌아올때도 NAT 을 통해서 External IP 의 Destination 이 Cluster 전용 IP 로 바뀌어서 전달됨. 
* POD 에서 외부와 통신한다면 NAT 에서 모두 패킷이 드랍된다. NAT 은 노드의 아이피만 알기 떄문이다. 
* 그래서 Masquerade 기법을 (SNAT) 통해서 POD 아이피를 NODE IP로 변경하는 작업이 필요하다. 

## Load Balancer 
* Service 의 Type 을 Load Balancer 로 세팅하면, 클러스터 아이피와 별개로 외부에서 접근가능한 IP 가 생성된다. 
* Load Balancer 는 노드들만 알고 POD 이 어떻게 배치되었는지는 모르기 때문에 노드별로 트레픽이 불균형하게 들어갈 수 있다. 
* 이를 맞추는 방법중에 하나가 iptable 을 가지고 하는 방법인데, 일단 Load Balancer 가 노드에게 트래픽을 보내면 노드에 설정된 iptable 정보를 통해서 다른 노드로 다시 전달  
* 이 과정에서 HOP 이 발생. 
* 이런 상황이 발생하니까 사용자가 잘 알아서 노드에 POD 을 분배해야함. 잘 알고 진행해야한다.


## Network policy
* POD 이나 서비스끼리 자유롭게 통신이 가능한데, 보안상의 정책등의 이유로 제어가 필요할때가 있다. 
* 예를 들어 일반적인 3-tier 어플리케이션의 경우 Frontend 가 다른 Frontend POD 에 접근하거나, Frontend 가 DB 에 접근하는 경우를 원천 봉쇄하고 싶을떄가 있다. 
* NetworkPolicy 를 통해서 세팅  


아주 만족스러운 좋은 영상입니다. 네트웍에 대한 가상화기술이나 iptable 등의 기술에 익숙하지 않아서 k8s 을 처음배울때 Service 개념이 진짜 혼란스러웠는데, 
이 영상을 보고 나서 큰 부분이 정리가 되었습니다. 
