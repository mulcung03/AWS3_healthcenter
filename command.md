## KUBECTL
객체 생성
kubectl create deployment nginx --image=nginx
객체 목록 불러오기
kubectl get all
kubectl get pod
kubectl get deploy
객체 추가정보 열람하기
kubectl get deploy -o wide
kubectl get pod -o wide
객체 상세 정보 확인하기
kubectl describe [pod/객체 이름]
kubectl describe [deploy/nginx]
Pod Status에 따른 Trouble Shooting
Status가 ImagePullBackOff/ ErrImagePull 일때,

kubectl describe [pod/객체 이름]
Status가 CrashLoopBackOff/Errors 일때,

kubectl logs [pod/객체 이름]
kubectl logs -f [pod/객체 이름]
Status가 Running이지만 서비스가 정상적이지 않을때,

kubectl exec -it [pod/객체 이름] -- /bin/bash
ls
df -k
curl localhost:80
exit
yaml을 통한 Pod 배포
apiVersion: v1
kind: Pod
metadata:
  name: declarative-pod
spec:
  containers:
  - name: memory-demo-ctr
    image: nginx
declarative-pod.yaml로 저장
kubectl create -f declarative-pod.yaml
##### 동일한 yaml 을 apply 명령으로도 실행
kubectl apply -f declarative-pod.yaml
객체 삭제하기
kubectl delete [객체 타입/객체 이름]
kubectl delete [객체타입,객체타입,…] --a
