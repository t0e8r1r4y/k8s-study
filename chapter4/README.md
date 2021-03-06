# 레플리케이션과 그 밖의 컨트롤러 : 관리되는 파드 배포  

파드를 직접 생성하고 배포하기보다는 디플로이먼트나 레플리카를 이용해서 관리한다.  

1. liveness probe  
      Pod의 스펙에 컨테이너의 liveness probe를 지정할 수 있다.
      쿠버네티스는 주기적으로 probe를 실행하고 probe가 실패하면 컨테이너를 다시 시작한다.  
      쿠버네티스가 probe를 실행하는 매커니즘
        - HTTP GET 프로브는 지정한 IP주소, 포트, 경로에 HTTP GET 요청을 수행한다. 프로브가 응답을 수신하고 응답코드가 오류를 나타내지 않는 경우 probe success  
        - 서버가 오류코드를 반환하거나 전혀 응답하지 않으면 프로브 실패로 간주 -> 컨테이너 재기동  
        - TCP 소켓 probe는 컨테이너의 지정된 포트에 TCP 연결을 시도  
        - 연결에 성공하면 probe가 성공한 것이고, 그렇지 않으면 컨테이너 재기동  
        - Exxec probe는 컨테이너 내의 임의의 명령을 실행하고 명령의 종료 상태 코드를 확인  
        - 상태코드가 0이면 probe success. 모든 다른 코드는 실패로 간주함  


     생성방법 -> scr -> kubia-liveness-probe.yaml 파일 실행
          kubectl describe po kubia-liveness 입력
          출력 로그에서
          Liveness:       http-get http://:8080/ delay=0s timeout=1s period=10s #success=1 #failure=3   
          책과 다르게 한번에 잘 실행 됨....
     
     동작상태 확인
     
     liveness probe의 추가 속성 설정  
     - kubectl describe 명령을 쳤을 때
     - Liveness: http-get http://:9090/ delay=0s timeout=1s period=10s #success=1 #failure=3 이런게 나올 수 있다.
     - 지연, 제 시간, 기간 등과 같은 추가속성을 지정가능
     - 해당 설정은 yaml 파일에서도 설정이 가능함.
     - 초기 지연을 설정하지 않으면 프로브가 컨테이너 시작과 동시에 실행되는데, 이 경우 대부분 애플리케이션이 요청을 받을 준비가 되어 있지 않아서 실패가 난다.
     - 이럴 경우 실패 횟수가 실패 임곗값을 초과하면 컨테이너가 재기동이 되므로
     - 반드시 애플리케이션 시작 시간을 고려해 초기 지연을 설정해야 한다는 점!!!


     효과적인 liveness probe 생성  
     - 특정 URL 경로에 요청하도록 probe를 구성해 애플리케이션 내에서 실행 중인 모든 주요 구성 요소가 살아 있는지 또는 응답이 없는지 확인하도록 구성이 가능하다.
     - HTTP 엔드포인트(/health)에 인증이 필요하지 않은지 확인이 필요. 그렇지 않느면 프로브가 항상 실패해서 컨테이너가 무한정으로 재시작 됨.
     - liveness probe는 애플리케이션 내부만 체크하고 외부 요인의 영향을 받지 않도록 해야함  
     - probe의 역할을 간단하게 만들어서 전체 성능에 영향을 주지 말아야함.
     - 컨테이너에서 자바 애플리케이션을 실행하는 경우 라이브니스 정보를 얻기 위해 Exec 프로브로 전체 JVM을 새로 기동하는 대신 HTTP GET liveness probe를 사용할 것. ( 기동절차에서 연산이 되기 때문)
     - probe 재시도 루프를 구현하지 말것. 알아서 재기동 됨.




2. 레플리케이션 컨트롤러 -> 파드의 생성과 관리를 맡아서 해줌

     레플리케이션 컨트롤러의 동작
     - 실행 중인 파드 목록을 지속적으로 모티너링하고 특정 "tpye"의 실제 파드 수가 의도하는 수와 일치하는지 항상 확인
     - 파드가 적게 실행되면 파드를 늘리고, 파드 실행 개수가 많으면 초과 복제본이 제거 됨
     - 레플리케이션컨트롤러는 엄밀히 파드 유형이 아니라 특정 레이블 셀렉터와 일치하는 파드 세트에 작동함


     컨트롤러 조정 루프  
     
     ![kube2 drawio](https://user-images.githubusercontent.com/91730236/157252822-c62525ab-177a-4adb-878c-5bd638614145.png)
     
     레플리케이션컨트롤러의 세 가지 요소
     - 레이블 셀렉터 ( Lable Selector )는 레플리케이션컨트롤러의 범위에 있는 파드를 결정
     - 레플리카 수 ( replica count )는 실행할 파드의 의도하는 수를 지정
     - 파드 템플릿 ( Pod template )은 새로운 파드 레플리카를 만들 때 사용
     - 레플리케이션컨트롤러의 레플리카 수, 레이블 렉렉터, 파드 템플릿은 언제든 수정이 됨. -> 단, 레플리카 수의 변경만 기존 파드에 영향
     - 레이블 셀렉터를 변경하면 기존 파드가 레플리케이션 컨트롤러의 범위를 벗어나므로 컨트롤러가 해당 파드에 대한 관리를 중지함
     - 레플레케이션컨트롤러는 파드를 생성한 후에는 파드의 실제 컨테이너 이미지, 환경 변수 및 기타 사항에 신경 안씀 -> 템플릿은 레플리케이션컨트롤러로 새 파드를 생성할 때만 영향


    레플리케이션컨트롤서 사용 시 이점
    - 기존 파드가 사라지면 새 파드를 시작해 파드가 항상 실행되도록 한다.
    - 클러스터 노드에 장애가 발생하면 장애가 발생한 노드에서 실행 중인 모든 파드에 관한 교체 복제본이 생성 됨
    - 수동 또는 자동으로 파드를 쉽게 수평 확장이 가능함


    레플리케이션 생성 예제
    - scr 추가
    - 컨트롤러가 삭제 된 파드를 재 생성하는 것은 삭제에 대한 대응이 아니라 부족한 파드 수에 대한 대응임. ( 삭제에 대한 통지는 API 서버가 수신하기는 함 )
            
            kubectl create -f kubia-rc.yaml
            replicationcontroller/kubia created
            
            kubectl get pod
            NAME              READY   STATUS              RESTARTS      AGE
            kubia-hb2zr       0/1     ContainerCreating   0             13s
            kubia-manual-v2   1/1     Running             2 (27m ago)   2d1h
            kubia-pbql5       1/1     Running             0             13s
            kubia-rhjzt       1/1     Running             0             13s
            
            kube terryakishin$ kubectl delete pod kubia-pbql5
            pod "kubia-pbql5" deleted
            
            kubectl get pod
            NAME              READY   STATUS    RESTARTS      AGE
            kubia-5mjgr       1/1     Running   0             66s
            kubia-hb2zr       1/1     Running   0             2m45s
            kubia-manual-v2   1/1     Running   2 (29m ago)   2d2h
            kubia-rhjzt       1/1     Running   0             2m45s


     레플리케이션컨트롤러의 범위 안팎으로 파드 이동
     - 레플리케이션컨트롤러가 생성한 파드는 어떤 식으로든 이 레플리케이션 컨트롤러와 묶이지 않는다.
     - 레플리케이션 컨트롤러는 레이블 셀렉터오 ㅏ일치하는 파드만을 관리함
     - 파드의 레이블을 변경하면 레플리케이션컨트롤러의 범위에서 제거되거나 추가될 수 있다. 한 레플리케이션컨트롤러에서 다른 레플리케이션 컨트롤러로 이동도 가능함
            
            kubectl label pod kubia-rhjzt app=foo --overwrite
            pod/kubia-rhjzt labeled
            
            kube terryakishin$ kubectl get pod
            NAME              READY   STATUS              RESTARTS      AGE
            kubia-5mjgr       1/1     Running             0             4m23s
            kubia-hb2zr       1/1     Running             0             6m2s
            kubia-manual-v2   1/1     Running             2 (33m ago)   2d2h
            kubia-ptmtf       0/1     ContainerCreating   0             4s
            kubia-rhjzt       1/1     Running             0             6m2s
            
            kube terryakishin$ kubectl get pods -L app
            NAME              READY   STATUS    RESTARTS      AGE     APP
            kubia-5mjgr       1/1     Running   0             4m32s   kubia
            kubia-hb2zr       1/1     Running   0             6m11s   kubia
            kubia-manual-v2   1/1     Running   2 (33m ago)   2d2h
            kubia-ptmtf       1/1     Running   0             13s     kubia
            kubia-rhjzt       1/1     Running   0             6m11s   foo     <- rc의 관리대상이 아닌 pod
            
     - 컨트롤러에서 파드를 제거 -> 특정 파드에 어떤 작업을 하려는 경우 해당 파드를 레플리케이션컨트롤러의 범위에서 제거하면 작업이 훨씬 간단함. 특히 장애가 발생했을 때 빼내서 따로 디버깅하면 편함
     - 레이블 셀렉터 변경


     파드 템플릿 변경
     
     수평 파드 스케일링
     - 레플리케이션컨트롤러 리소스의 replicas 필드 값을 변경하기만 하면 됨
     - kubectl scale rc kubia --replicas=10

            kube terryakishin$ kubectl get pods
            NAME              READY   STATUS    RESTARTS      AGE
            kubia-5mjgr       1/1     Running   0             7m29s
            kubia-hb2zr       1/1     Running   0             9m8s
            kubia-hhpdq       1/1     Running   0             39s
            kubia-hnr9h       1/1     Running   0             39s
            kubia-manual-v2   1/1     Running   2 (36m ago)   2d2h
            kubia-nw65q       1/1     Running   0             39s
            kubia-p2wsc       1/1     Running   0             39s
            kubia-plvkx       1/1     Running   0             39s
            kubia-ptmtf       1/1     Running   0             3m10s
            kubia-rhjzt       1/1     Running   0             9m8s
            kubia-rvvln       1/1     Running   0             39s
            kubia-swnxx       1/1     Running   0             39s
            
            kube terryakishin$ kubectl scale rc kubia --replicas=2
            replicationcontroller/kubia scaled
            
            kube terryakishin$ kubectl get pods
            NAME              READY   STATUS        RESTARTS      AGE
            kubia-5mjgr       1/1     Running       0             8m14s
            kubia-hb2zr       1/1     Running       0             9m53s
            kubia-hhpdq       1/1     Terminating   0             84s
            kubia-hnr9h       1/1     Terminating   0             84s
            kubia-manual-v2   1/1     Running       2 (37m ago)   2d2h
            kubia-nw65q       1/1     Terminating   0             84s
            kubia-p2wsc       1/1     Terminating   0             84s
            kubia-plvkx       1/1     Terminating   0             84s
            kubia-ptmtf       1/1     Terminating   0             3m55s
            kubia-rhjzt       1/1     Running       0             9m53s
            kubia-rvvln       1/1     Terminating   0             84s
            kubia-swnxx       1/1     Terminating   0             84s


     레플리케이션컨트롤러 삭제
     - 레플리케이션컨트롤러를 삭제하면 파드도 삭제. 그러마 rc만 삭제하고 pod는 실행 상태로 둘 수 있음
     - kubectl delete rc kubia --cascade=fals <- 이 옵션을 줘야함


3. 레플리케이션컨트롤러(초기) VS 레플리카셋(이후)

     레플리카셋
     - rc에 비해서 표현식이 풍부함 -> 특정 레이블이 없는 파드나 레이블의 값과 상관없이 특정 레이블의 키를 갖는 파드를 매칭 시킬 수 있음.
     - 


4. 데몬셋의 사용
     클러스터의 모든 노드에, 노드당 하나의 파드만 실행되기를 원하는 경우 -> 시스템 수준의 작업을 수행하는 인프라 관련 파드
     
     데몬셋으로 모든 노드에 파드 실행하기 -> 데몬셋 오브젝트를 생성해야 함
     
     
5. 완료 가능한 단일 태스크를 수행하는 파드 실행 ( 잡리소스 = 배치 )

     쿠버네티스틑 job 리소스로 이런 기능을 지원함 -> 파드의 컨테이너 내부에서 실행 중인 프로세스가 성공적으로 완료되면 컨테이너를 다시 시작하지 않는 파드를 실행 가능
     잡 리소스를 정의하고 예제 실행
     
     잡에서 여러 파드 인스턴스 실행하기
     - 잡은 두 개 이상의 파드 인스턴스를 생성해 병렬 또는 순차적으로 실행하도록 구성이 가능하다. ( job spec에서 completions와 parallelism 속성 설정 )
     - 잡을 스케일링하고 싶으면 parallelism을 증가시켜 주면 다른 파드가 즉시 기동 됨.
     - 잡 파드가 완료되는 데 걸리는 시간을 제한하고 싶으면 activeDeadlineSeconds 속성을 설정하여 실행 시간을 제한할 수 있음.
     - 잡은 스케쥴링이 가능함 ( 주기적 혹은 한번만 )-> 이걸 크론잡(CronJob)이라고 함






-------------------------------------
#요약
- 컨테이너가 더 이상 정상적이지 않으면 즉시 쿠버네티스가 컨테이너를 다시 시작하도록 라이브니스 프로브를 지정할 수 있다.
- 파드는 실수로 삭제되거나 실행 중인 노드에 장애가 발생하거나 노드에서 퇴출되면 다시 생성되지 않기 때문에 파드를 직접 생성하면 안된다.
- 레플리케이션컨트롤러는 의도하는 수의 파드 복제본을 항상 실행 상태로 유지한다.
- 파드를 수평으로 스케일링하려면, 쉽게  레플리케이션컨트롤러에 의도하는 레플리카 수를 변경하는 것만으로도 가능하다.
- 파드는 레플리케이션 컨트롤러가 소유하지 않으며, 필요한 경우 레플리케이션컨르롤러 간에 이동할 수 있다.
- 레플리케이션컨트롤러는 파드 템플릿에서 새로운 파드를 생성한다. 템플릿을 변경해도 기존의 파드에는 영향을 미치지 않는다.
- 레플리케이션컨트롤러는 레플리카셋과 디플로이먼트로 교체해야 하며, 레플리카셋과 디플로이먼트는 동일한 기능을 제공하면서 추가적인 강력한 기능을 제공한다.
- 레플리케이션컨트롤러와 레플리카셋은 임의의 클러스터 노드에 파드를 스케줄링하는 반면, 데몬셋은 모든 노드에 데몬셋이 정의한 파드의 인스턴스 하나만 실행되도록 한다.
- 배치 작업을 수행하는 파드는 쿠버네티스의 잡 리소스로 생성해야 하며, 직접 생성하거나 레플리케이션컨트롤러 또는 유사한 오프벡트로 생성하지 말아야 한다.
- 나중에 언젠가 실행해야 하는 잡은 크론잡 리소스를 통해 생성할 수 있다.,





