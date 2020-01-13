# aws eks

{% embed url="https://qiita.com/Kta-M/items/3b99b849ad777c5dfd29" %}

```bash
$ eksctl version
[ℹ]  version.Info{BuiltAt:"", GitCommit:"", GitTag:"0.12.0"}
$ aws --version
aws-cli/1.17.0 Python/3.7.4 Darwin/19.2.0 botocore/1.14.0
$ kubectl version --client --short
Client Version: v1.17.0
```

```
$ eksctl create cluster --name nky-guestbook --fargate
```

command result

![](.gitbook/assets/sukurnshotto-2020-01-13-163300.png)

aws eks cluster

![](.gitbook/assets/sukurnshotto-2020-01-13-163348.png)

```bash
$ eksctl utils associate-iam-oidc-provider --region=ap-northeast-1 --cluster=nky-guestbook --approve
```

![](.gitbook/assets/sukurnshotto-2020-01-13-163519.png)

![](.gitbook/assets/sukurnshotto-2020-01-13-163543.png)

```bash
$ eksctl create iamserviceaccount --name alb-ingress-controller \
  --namespace kube-system \
  --cluster nky-guestbook \
  --attach-policy-arn arn:aws:iam::260859027812:policy/ALBIngressControllerIAMPolicy \
  --approve --override-existing-serviceaccounts
```

![](.gitbook/assets/sukurnshotto-2020-01-13-164126.png)

![](.gitbook/assets/sukurnshotto-2020-01-13-164155.png)

```bash
$ kubectl get sa -n kube-system alb-ingress-controller -o jsonpath="{.metadata.annotations['eks\.amazonaws\.com/role-arn']}"
arn:aws:iam::260859027812:role/eksctl-nky-guestbook-addon-iamserviceaccount-Role1-M1786EM0W8UP
```

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/rbac-role.yaml
```

![](.gitbook/assets/sukurnshotto-2020-01-13-164454.png)

{% code title="alb-ingress-controller.yaml" %}
```bash
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: alb-ingress-controller
  name: alb-ingress-controller
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: alb-ingress-controller
  template:
    metadata:
      labels:
        app.kubernetes.io/name: alb-ingress-controller
    spec:
      serviceAccountName: alb-ingress-controller
      containers:
        - name: alb-ingress-controller
          image: docker.io/amazon/aws-alb-ingress-controller:v1.1.4
          args:
            - --ingress-class=alb
            - --cluster-name=nky-guestbook # クラスタ名
            - --aws-region=ap-northeast-1
            - --aws-vpc-id=vpc-018a3914f392602b6 # eksctlで作成されたVPCのid
          resources: {}
```
{% endcode %}

```bash
$ kubectl apply -f alb-ingress-controller.yaml
deployment.apps/alb-ingress-controller created
```

```bash
$ git clone git@github.com:kubernetes/examples.git
```

Serviceのタイプを`ClusterIP`にします。

{% code title="examples/guestbook/frontend-service.yaml" %}
```bash
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # comment or delete the following line if you want to use a LoadBalancer
  # type: NodePort
  # if your cluster supports it, uncomment the following to automatically create
  # an external load-balanced IP for the frontend service.
  # type: LoadBalancer
  type: ClusterIP
  ports:
  - port: 80
  selector:
    app: guestbook
    tier: frontend
```
{% endcode %}

```bash
$ kubectl apply -f examples/guestbook/
```

![](.gitbook/assets/sukurnshotto-2020-01-13-165005.png)

{% code title="nginx.yaml" %}
```bash
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: nginx
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /*
            backend:
              serviceName: frontend
              servicePort: 80
```
{% endcode %}

デプロイ！

```bash
$ kubectl apply -f nginx.yaml
ingress.networking.k8s.io/nginx created
```

```bash
$ kubectl logs -n kube-system $(kubectl get po -n kube-system -o name | grep alb | cut -d/ -f2) -f
-------------------------------------------------------------------------------
AWS ALB Ingress controller
  Release:    v1.1.4
  Build:      git-0db46039
  Repository: https://github.com/kubernetes-sigs/aws-alb-ingress-controller.git
-------------------------------------------------------------------------------

W0113 07:48:23.929712       1 client_config.go:549] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
I0113 07:48:24.028495       1 controller.go:121] kubebuilder/controller "level"=0 "msg"="Starting EventSource"  "controller"="alb-ingress-controller" "source"={"Type":{"metadata":{"creationTimestamp":null}}}
I0113 07:48:24.028750       1 controller.go:121] kubebuilder/controller "level"=0 "msg"="Starting EventSource"  "controller"="alb-ingress-controller" "source"={"Type":{"metadata":{"creationTimestamp":null},"spec":{},"status":{"loadBalancer":{}}}}
I0113 07:48:24.028813       1 controller.go:121] kubebuilder/controller "level"=0 "msg"="Starting EventSource"  "controller"="alb-ingress-controller" "source"=
I0113 07:48:24.028971       1 controller.go:121] kubebuilder/controller "level"=0 "msg"="Starting EventSource"  "controller"="alb-ingress-controller" "source"={"Type":{"metadata":{"creationTimestamp":null},"spec":{},"status":{"loadBalancer":{}}}}
I0113 07:48:24.029012       1 controller.go:121] kubebuilder/controller "level"=0 "msg"="Starting EventSource"  "controller"="alb-ingress-controller" "source"=
I0113 07:48:24.029131       1 controller.go:121] kubebuilder/controller "level"=0 "msg"="Starting EventSource"  "controller"="alb-ingress-controller" "source"={"Type":{"metadata":{"creationTimestamp":null}}}
I0113 07:48:24.029421       1 controller.go:121] kubebuilder/controller "level"=0 "msg"="Starting EventSource"  "controller"="alb-ingress-controller" "source"={"Type":{"metadata":{"creationTimestamp":null},"spec":{},"status":{"daemonEndpoints":{"kubeletEndpoint":{"Port":0}},"nodeInfo":{"machineID":"","systemUUID":"","bootID":"","kernelVersion":"","osImage":"","containerRuntimeVersion":"","kubeletVersion":"","kubeProxyVersion":"","operatingSystem":"","architecture":""}}}}
I0113 07:48:24.029863       1 leaderelection.go:205] attempting to acquire leader lease  kube-system/ingress-controller-leader-alb...
I0113 07:48:24.060083       1 leaderelection.go:214] successfully acquired lease kube-system/ingress-controller-leader-alb
I0113 07:48:24.162235       1 controller.go:134] kubebuilder/controller "level"=0 "msg"="Starting Controller"  "controller"="alb-ingress-controller"
I0113 07:48:24.262543       1 controller.go:154] kubebuilder/controller "level"=0 "msg"="Starting workers"  "controller"="alb-ingress-controller" "worker count"=1
E0113 07:57:31.003554       1 controller.go:217] kubebuilder/controller "msg"="Reconciler error" "error"="no object matching key \"default/nginx\" in local store"  "controller"="alb-ingress-controller" "request"={"Namespace":"default","Name":"nginx"}
I0113 07:57:34.629321       1 security_group.go:36] default/nginx: creating securityGroup f314e9ea-default-nginx-ef8b:managed LoadBalancer securityGroup by ALB Ingress Controller
I0113 07:57:34.698577       1 tags.go:69] default/nginx: modifying tags {  ingress.k8s.aws/cluster: "nky-guestbook",  ingress.k8s.aws/stack: "default/nginx",  ingress.k8s.aws/resource: "ManagedLBSecurityGroup",  kubernetes.io/cluster-name: "nky-guestbook",  kubernetes.io/namespace: "default",  kubernetes.io/ingress-name: "nginx"} on sg-0e206bef6a39cb298
I0113 07:57:34.856703       1 security_group.go:75] default/nginx: granting inbound permissions to securityGroup sg-0e206bef6a39cb298: [{    FromPort: 80,    IpProtocol: "tcp",    IpRanges: [{        CidrIp: "0.0.0.0/0",        Description: "Allow ingress on port 80 from 0.0.0.0/0"      }],    ToPort: 80  }]
I0113 07:57:35.200731       1 loadbalancer.go:191] default/nginx: creating LoadBalancer f314e9ea-default-nginx-ef8b
I0113 07:57:36.342095       1 loadbalancer.go:208] default/nginx: LoadBalancer f314e9ea-default-nginx-ef8b created, ARN: arn:aws:elasticloadbalancing:ap-northeast-1:260859027812:loadbalancer/app/f314e9ea-default-nginx-ef8b/7ea74275497c1f2c
I0113 07:57:36.522857       1 targetgroup.go:119] default/nginx: creating target group f314e9ea-a0a02c491ab46968cae
I0113 07:57:36.679487       1 targetgroup.go:138] default/nginx: target group f314e9ea-a0a02c491ab46968cae created: arn:aws:elasticloadbalancing:ap-northeast-1:260859027812:targetgroup/f314e9ea-a0a02c491ab46968cae/2e657198b59f7daa
I0113 07:57:36.704401       1 tags.go:43] default/nginx: modifying tags {  ingress.k8s.aws/stack: "default/nginx",  kubernetes.io/cluster/nky-guestbook: "owned",  kubernetes.io/service-name: "frontend",  kubernetes.io/service-port: "80",  ingress.k8s.aws/resource: "default/nginx-frontend:80",  kubernetes.io/namespace: "default",  kubernetes.io/ingress-name: "nginx",  ingress.k8s.aws/cluster: "nky-guestbook"} on arn:aws:elasticloadbalancing:ap-northeast-1:260859027812:targetgroup/f314e9ea-a0a02c491ab46968cae/2e657198b59f7daa
I0113 07:57:36.826879       1 targets.go:80] default/nginx: Adding targets to arn:aws:elasticloadbalancing:ap-northeast-1:260859027812:targetgroup/f314e9ea-a0a02c491ab46968cae/2e657198b59f7daa: 192.168.145.15:80, 192.168.164.170:80, 192.168.181.91:80
I0113 07:57:37.088179       1 listener.go:110] default/nginx: creating listener 80
I0113 07:57:37.141662       1 rules.go:60] default/nginx: creating rule 1 on arn:aws:elasticloadbalancing:ap-northeast-1:260859027812:listener/app/f314e9ea-default-nginx-ef8b/7ea74275497c1f2c/95135c054d705e9a
I0113 07:57:37.164508       1 rules.go:77] default/nginx: rule 1 created with conditions [{    Field: "path-pattern",    Values: ["/*"]  }]
I0113 07:57:37.714012       1 instance_attachment_v2.go:192] default/nginx: granting inbound permissions to securityGroup sg-02979cc22d6b1d259: [{    FromPort: 0,    IpProtocol: "tcp",    ToPort: 65535,    UserIdGroupPairs: [{        GroupId: "sg-0e206bef6a39cb298"      }]  }]
I0113 07:57:38.658086       1 rules.go:82] default/nginx: modifying rule 1 on arn:aws:elasticloadbalancing:ap-northeast-1:260859027812:listener/app/f314e9ea-default-nginx-ef8b/7ea74275497c1f2c/95135c054d705e9a
I0113 07:57:38.681712       1 rules.go:98] default/nginx: rule 1 modified with conditions [{    Field: "path-pattern",    Values: ["/*"]  }]
W0113 07:58:16.333409       1 reflector.go:270] pkg/mod/k8s.io/client-go@v0.0.0-20181213151034-8d9ed539ba31/tools/cache/reflector.go:95: watch of *v1.Secret ended with: too old resource version: 2395 (5579)

```

```bash
$ kubectl get ing
NAME    HOSTS   ADDRESS                                                                  PORTS   AGE
nginx   *       f314e9ea-default-nginx-ef8b-469930252.ap-northeast-1.elb.amazonaws.com   80      115s
```

![](.gitbook/assets/sukurnshotto-2020-01-13-170004.png)

## Kubernetes ウェブ UI \(ダッシュボード\) のデプロイ <a id="dashboard-tutorial"></a>

{% hint style="info" %}
[https://docs.aws.amazon.com/ja\_jp/eks/latest/userguide/dashboard-tutorial.html](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/dashboard-tutorial.html)
{% endhint %}

```bash
$ kubectl apply -f metrics-server-0.3.6/deploy/1.8+/
```

![](.gitbook/assets/sukurnshotto-2020-01-13-170345.png)

```bash
$ kubectl get deployment metrics-server -n kube-system
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   1/1     1            1           79s
```

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created
```

{% code title="eks-admin-service-account.yaml" %}
```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eks-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: eks-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: eks-admin
  namespace: kube-system
```
{% endcode %}

```bash
$ kubectl apply -f eks-admin-service-account.yaml
serviceaccount/eks-admin created
clusterrolebinding.rbac.authorization.k8s.io/eks-admin created
```

```bash
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
Name:         eks-admin-token-89m9b
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: eks-admin
              kubernetes.io/service-account.uid: a141acf9-35db-11ea-815b-0a6aa20a1ce8

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJla3MtYWRtaW4tdG9rZW4tODltOWIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZWtzLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYTE0MWFjZjktMzVkYi0xMWVhLTgxNWItMGE2YWEyMGExY2U4Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmVrcy1hZG1pbiJ9.K38-z8Pam-J-iLfrBgR3TuKE6r_IGolXBt92Pa_JDbXTgOTcD6HcGgvfC_sIGiy-3Oqzc-YtFbPSPoOUOib5htWLLYJB33jGamdIGeM_cUGurKgB2f9Fo83EORyShJ-C8RZalH9WlCJ81wHBbTVJDVOnHlYmzJdGp03g3xQme7KqcFYsEN-J-N7BThZzParY5NgrVPHm-PXlzzjbz0UJlcUYrKWoIOPOe731QAieT37vORrYbE6IuATcxj4qyg6jE9uLvClrLi0izgYBo9hrJvp-rUsdLWDaIrZx5EimVVKi-qihI9q6Up_I1saVbUaMrP75bgcfmtfdaGKNJI7NCg

```

```bash
$ kubectl proxy
Starting to serve on 127.0.0.1:8001

```

```bash
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login
```

![](.gitbook/assets/sukurnshotto-2020-01-13-175442.png)

![](.gitbook/assets/sukurnshotto-2020-01-13-175500.png)

**eks cluster delete**

```bash
$ eksctl delete cluster --name nky-guestbook
```

参考サイト

{% embed url="https://839.hateblo.jp/entry/2019/12/08/172020" %}

[https://github.com/pahud/amazon-eks-workshop/tree/master/eks-fargate](https://github.com/pahud/amazon-eks-workshop/tree/master/eks-fargate)



