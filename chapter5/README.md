# 서비스: 클라이언트가 파드를 검색하고 통신을 가능하게 함

1. 서비스란? 파드가 다른 파드를 찾는 방법

       서비스가 필요한 이유
       - 파드는 일시적이다. 파드가 다른 파드를 위한 공간을 확보하려고 노드에서 제거되거나, 누군가 파드 수를 줄이거나, 클러스터 노드의 장애로 언제든 다른 노드로 이동할 수 있다.
       - 쿠버네티스는 노드에 파드를 스케줄링한 후 파드가 시작되기 바로 전의 파드의 IP 주소를 할당한다. 따라서 클라이언트는 서버 pod의 IP 주소를 미리 할 수 없다.
       - 수평 스케일링은 여러 파드가 동일한 서비스를 제공할 수 있음을 의미함. 각 파드는 고유한 IP 주소가 있다. 클라이언트는 서비스를 지원하는 파드의 수와 IP에 상관하지 않아야 한다.
       - 파드는 개별 IP 목록을 유지할 필요가 없다. 그 대신 모든 파드는 단일 IP 주소로 액세스 할 수 있어야 한다.  


       쿠버네티스의 서비스는 동일한 서비스를 제공하는 파드 그룹에 지속적인 단일 접점을 만들려고 할 때 생성하는 리소스.
       각 서비스는 서비스가 존재하는 동안 절대 바뀌지 않는 IP 주소와 포트가 있다.
       클라이언트는 해당 IP와 포트로 접속한 다음 해당 서비스를 지원하는 파드 중 하나로 연결 된다.
       클라이언트는 서비스를 제공하는 개별 파드의 위치를 알 필요 없으므로, 이 파드는 언제든지 클러스터 안에서 이동할 수 있다.
       
<center><img src="https://user-images.githubusercontent.com/91730236/157878478-be84aab6-6cf8-460e-b611-a41c064d110d.png" width="700" height="350"></center>      

       
       
       서비스의 생성 -> 서비스 연결은 서비스 뒷단의 모든 파드로 로드밸런싱 된다.
       레이블 셀렉터는 어떤 파드가 서비스에 속하는지 정한다.
       
<center><img src="https://user-images.githubusercontent.com/91730236/157879122-c8700e1a-9b59-4b11-88c0-5ea45d2d3aed.png" width="700" height="350"></center>


       kubectl expose로 서비스 생성 -> expose를 사용해서 레플리케이션컨트롤러를 노출하는데 사용
       
       src에서 kubia-svc.yaml create 후 get 조회  
       
       kubectl get svc
       NAME               TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
       flink-jobmanager   ClusterIP      10.109.242.196   <none>        6123/TCP,6124/TCP,8081/TCP   61d
       kubernetes         ClusterIP      10.96.0.1        <none>        443/TCP                      161d
       kubia              ClusterIP      10.96.166.187    <none>        90/TCP                       10s
       kubia-http         LoadBalancer   10.100.233.74    <pending>     9090:31813/TCP               9d
       
       
       
       클러스터 내에서 서비스 테스트
       - 확실한 방법은 서비스의 클러스터 IP로 요청을 보내고 응답을 로그로 남기는 파드를 만드는 것.
       - 그런 다음 파드의 로그를 검사해 서비스의 응답이 무엇인지 확인할 수 있다.
       - 쿠버네티스 노드로 ssh로 접속하고 curl 명령을 실행 할 수 있다.
       - kubectl exec 명령어로 기존 파드에서 curl 명령을 실행할 수 있다.
       
       kubectl exec kubia-kkhk9 -- curl -s http://10.96.166.187:90
       You've hit kubia-tbg58
       
       동작순서
![kube_service drawio (2)](https://user-images.githubusercontent.com/91730236/157883721-5c1aeb39-e18b-4583-a648-e6662f266832.png)

       
       서비스의 세션 어피니티 구성 : 특정 클라이언트의 모든 요청을 매번 같은 파드로 리디렉션 하려면 spec: 속성에서 sessionAffinity: ClientIP라고 설정
       동일한 서비스에서 여러 개의 포트 노출 : 여러 포트가 있는 서비스를 만들 때는 각 포트의 이름을 지정해야 한다. 
       이름이 지정된 포트를 사용할 수 있다. -> src 에서 ~muilt.yaml 실행하고 위 방식으로 테스트
       
       
       서비스 검색 방법 ( 요부분 조금 더 이해를 할 필요가 있다 )
       - 환경변수를 통한 서비스 검색 kubectl exec kubia-rx2vs env
       - DNS를 통한 서비스 검색
       - FQDN을 통한 서비스 연결
       - 파드의 컨테이너 내에서 셸 실행
     
     
     
---------------------------     
2. 클러스터 외부에 있는 서비스 연결  
       
       서비스 엔드포인트
       kubectl describe svc kubia
       Name:              kubia
       Namespace:         default
       Labels:            <none>
       Annotations:       <none>
       Selector:          app=kubia
       Type:              ClusterIP
       IP Family Policy:  SingleStack
       IP Families:       IPv4
       IP:                10.100.218.171
       IPs:               10.100.218.171
       Port:              http  80/TCP
       TargetPort:        8080/TCP
       Endpoints:         172.17.0.10:8080,172.17.0.8:8080,172.17.0.9:8080
       Port:              https  90/TCP
       TargetPort:        9090/TCP
       Endpoints:         172.17.0.10:9090,172.17.0.8:9090,172.17.0.9:9090
       Session Affinity:  None
       Events:            <none>
       
       kubectl get endpoints kubia
       NAME    ENDPOINTS                                                      AGE
       kubia   172.17.0.10:9090,172.17.0.8:9090,172.17.0.9:9090 + 3 more...   10m
       
       
       서비스는 파드에 직접 연결되지 않는다. 엔드포인트 리소스가 그 사이에 있다.
       파드셀렉터는 서비스 스펙에 정의되어 있지만, 들어오는 연결을 전달할 때 직접 사용하지 않는다.
       대신 셀렉터는 IP와 포트 목록을 작성하는데 사용되며 엔드포인트 리소스에 저장
       클라이언트가 서비스에 연결하면
       서비스 프록시는 이들 중 하나의 IP와 포트 쌍을 선택하고 들어온 연결을 대상 파드의 수신 대기 서버로 전달.
       
       
       그래서 -> 서비스 엔드포인트는 수동으로 구성이 가능하다.
       1. 셀렉터가 없는 서비스 생성
       2. 엔드포인트는 별도의 리소스이며, 서비스 속성은 아님. -> 엔드포인트 자동 생성 안됨 -> 엔드포인트 서비스를 생성해야 한다.
       3. 엔드포인트 오브젝트는 서비스와 이름이 같아야 하고 서비스를 제공하는 대상 IP 주소와 포트 목록을 가져야 한다.
       4. 서비스와 엔드포인트 리소스가 모두 서버에 게시되면 파드 셀렉터가 있는 서비스처럼 서비스를 사용할 수 있다.
       5. 서비스가 만들어진 후 만들어진 컨테이너에는 서비스의 환경변수가 포함되며 IP:포드 쌍에 대한 모든 연결은 서비스 엔드포인트 간에 로드밸런싱한다.
       ( external-service-endpoints.yaml )

![kube_service drawio (3)](https://user-images.githubusercontent.com/91730236/157888958-5e09d0b3-4f5c-4aed-9474-7cd7f7a427df.png)



       외부 서비스를 위한 별칭 생성
       - 서비스의 엔드포인트를 수동으로 구성해 외부 서비스를 노출하는 대신 좀 더 간단한 방법으로 FQDN(Fully Qualified Domain Name)으로 외부 서비스를 참조 할 수 있다.
       - 외부 서비스의 별칭으로 사용되는 서비스를 만들려면 유형 필드는 ExternalName으로 설정해 서비스 리소르를 만든다.
       - 서비스가 생성되면 파드는 서비스의 FQDN을 사용하는 대신 external-service.default.svc.cluster.local 도메인 이름으로 외부 서비스에 연결 가능
       - 서비스를 사용하는 파드에서 실제 서비스 이름과 위치가 숨겨져 나중에 externalName 속성을 변경하거나 유형을 다시 ClusterIP로 변경하고 서비스 스펙을 만들어 서비스 스펙을 수정하면 나중에 다른 서비스를 가리킬 수 없다.
       - 서비스를 위한 엔드포인트 오브젝트를 수동 혹은 서비스에 레이블 셀렉터를 지정해 엔드포인트가 자동으로 생성되도록 한다.
       - ExternalName 서비스는 DNS 레벨에서만 구현된다.
       - 서비스에 관한 간단한 CNAME DNS 레코드가 생성된다.
       - 서비스에 연결하는 클라이언트는 서비스 프록시를 완전히 무시하고 외부 서비스에 직접 연결된다.
       - CNAME 레코드는 IP 주소 대신 FQDN을 가리킨다.

        


---------------------------
3. 클러스터 외부에 있는 서비스 연결

       외부 클라이언트 서비스 노출 방법
       - 노드포트로 서비스 유형 설정: 노드포트 서비스의 경우 각 클러스터 노드는 노드 자체에서 포트를 열고 해당 포트로 수신된 트래픽을 서비스로 전달한다.
                               이 서비스는 내부 클러스터 IP와 포트로 액세스할 수 있을 뿐만 아니라 모든 노드의 전용 포트로도 액세스 할 수 있다.
       - 서비스 유형을 노드포트 유형의 확장인 로드밸런서로 설정 : 쿠버네티스가 실행 중인 클라우드 인프라에서 프로비저닝 된 전용 로드밸런서로 서비스에 액세스할 수 있다.
       - 단일 IP 주소로 여러 서비스를 노출하는 인그레스 리소스 만들기 : HTTP 레벨에서 작동하므로 4계층 서비스보다 더 많은 기능을 제공할 수 있다. 인그레스 리소스



 >     노드포트 서비스 생성 및 확인
 >     kubectl get svc kubia-nodeport
 >     NAME             TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
 >     kubia-nodeport   NodePort   10.104.45.206   <none>        80:30123/TCP   20s

![kube_service drawio (4)](https://user-images.githubusercontent.com/91730236/157894377-03592961-0458-4299-b9f1-c35c12f67546.png)

       
       
>      외부 로드밸런서로 서비스 노출 ( minikube는 로드밸런서 제공 안함 )
>      -클라우드 공급자에서 실행되는 쿠버네티스 클러스터는 일반적으로 클라우드 인프라에서 로드밸런서를 자동으로 프로비저닝하는 기능을 제공한다.
>      -노드포트 대신 서비스 유형을 로드밸런서로 설정하기만 하면 된다.
>      -로드밸런서는 공개적으로 액세스 가능한 고유한 IP 주소를 가지며 모든 연결을 서비스로 전달한다.
>      -따라서 로드밸런서의 IP 주소로 서비스에 엑세스 할 수 있다.
>      -로드밸런서 서비스를 지원하지 않는 환경에서 실행 중인 경우 로드밸런서는 프로비저닝되지 않지만 서비스는 여전히 노드포트 서비스처럼 작동
>      -로드밸런서 서비스는 노드포트 서비스의 확장이기 

![kube_service drawio (5)](https://user-images.githubusercontent.com/91730236/157895288-f3e95673-3f52-474d-ad94-bbe8b14faf5d.png)

       
       외부 연결 특성의 이해
       -불필요한 네트워크 홉의 이해와 예방
       -외부 클라이언트가 노드포트로 서비스에 접속할 경우 임의로 선택된 파드가 연결을 수신한 동일한 노드에서 실행 중일 수도 있고 그렇지 않을 수도 있음
       -파드에 도달하려면 추가적인 네트워크 홉이 필요할 수 있음
       -근데 이게 항상 바람직하지 않음
       -외부의 연결을 수신한 노드에서 실행 중인 파드로만 외부 트래픽을 전달하도록 서비스를 구성해 이 추가 홉을 방지할 수 있다.
       -서비스 spec: 에 externalTrafficPolicy: Local로 설정
       -서비스 정의에 이 설정이 포함돼 있고 서비스의 노드포트로 외부 연결이 열린 경우 서비스 프록시는 로컬에 실행 중인 파드를 선택한다.
       -로컬파드가 존재하지 않으면 연결이 중단된다.
       -어노테이션을 사용하지 않을 때 연결이 임의의 글로벌 파드로 전달되지 않음.
       -따라서 로드밸런서는 파드가 하나 이상 있는 노드에만 연결을 전달하도록 해야 한다.
       -클라이언트 IP가 보존되지 않음 인식

       
       
       
       
----------------------------------------
4. 외부 클라이언트에 서비스 노출
       
  인그레스란? 들어가거나 들어가는 행위. 들어갈 권리. 들어갈 수단이나 장소, 진입로  
  인그레스가 필요한 이유 -> 인그레스 하나로 수십 개의 서비스에 접근이 가능하도록 지원. 인그레스는 네트워크 스택의 애플리케이션 계층(HTTP)에서 작동함 서비스가 할 수 없는 쿠키 기반 세션 어피니티 등과 같은 기능 제공
       
![스크린샷 2022-03-12 오전 7 50 14](https://user-images.githubusercontent.com/91730236/157984966-1f1caf12-4c9d-4a5b-be9f-77c70530ed97.png)


              minikube addons enable ingress
                  ▪ Using image k8s.gcr.io/ingress-nginx/controller:v1.0.0-beta.3
                  ▪ Using image k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.0
                  ▪ Using image k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.0
              🔎  Verifying ingress addon...
              🌟  'ingress' 애드온이 활성화되었습니다
       
              kubectl get po --all-namespaces
              NAMESPACE          NAME                                        READY   STATUS              RESTARTS        AGE
              custom-namespace   kubia-manual                                1/1     Running             3 (3d ago)      5d2h
              default            kubia-g2rk4                                 1/1     Running             0               73m
              default            kubia-rx2vs                                 1/1     Running             0               73m
              default            kubia-skffm                                 1/1     Running             0               73m
              ingress-nginx      ingress-nginx-admission-create--1-slms7     0/1     Completed           0               8m9s
              ingress-nginx      ingress-nginx-admission-patch--1-tnmz6      0/1     Completed           1               8m9s
              ingress-nginx      ingress-nginx-controller-69bdbc4d57-7nhhz   1/1     Running             0               8m9s
              istio-system       istio-ingressgateway-8dbb57f65-dkv28        0/1     ContainerCreating   0               125d
              istio-system       istiod-7859559dd-tsq74                      0/1     Pending             0               125d
              kube-system        coredns-78fcd69978-jvc8r                    1/1     Running             16 (3d ago)     161d
              kube-system        etcd-minikube                               1/1     Running             17 (3d ago)     161d
              kube-system        kube-apiserver-minikube                     1/1     Running             17 (3d ago)     161d
              kube-system        kube-controller-manager-minikube            1/1     Running             17 (3d ago)     161d
              kube-system        kube-proxy-tmfgd                            1/1     Running             16 (3d ago)     161d
              kube-system        kube-scheduler-minikube                     1/1     Running             17 (3d ago)     161d
              kube-system        storage-provisioner                         1/1     Running             45 (100m ago)   161d

------------------------------
요약! ( 외부와 통신이 결정되는 부분 -> 제대로 학습 필요 )
