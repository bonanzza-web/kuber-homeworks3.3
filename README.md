# Домашнее задание к занятию «Как работает сеть в K8s»

### Цель задания

Настроить сетевую политику доступа к подам.

### Чеклист готовности к домашнему заданию

1. Кластер K8s с установленным сетевым плагином Calico.

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Документация Calico](https://www.tigera.io/project-calico/).
2. [Network Policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/).
3. [About Network Policy](https://docs.projectcalico.org/about/about-network-policy).

-----

### Задание 1. Создать сетевую политику или несколько политик для обеспечения доступа

1. Создать deployment'ы приложений frontend, backend и cache и соответсвующие сервисы.
2. В качестве образа использовать network-multitool.
3. Разместить поды в namespace App.
4. Создать политики, чтобы обеспечить доступ frontend -> backend -> cache. Другие виды подключений должны быть запрещены.
5. Продемонстрировать, что трафик разрешён и запрещён.


------

### Ответы:

Данная работа выполнена на kubespray.    

Применяем манифесты:    

```
ubuntu@node-1:~/deploys$ kubectl apply -f frontend.yml
deployment.apps/frontend created
service/frontend-svc unchanged
ubuntu@node-1:~/deploys$ kubectl apply -f backend.yml
deployment.apps/backend created
service/backend-svc created
ubuntu@node-1:~/deploys$ kubectl apply -f cache.yml
deployment.apps/cache created
service/cache-svc created
ubuntu@node-1:~/deploys$ kubectl get deploy
No resources found in default namespace.
ubuntu@node-1:~/deploys$ kubectl get deploy -n app
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
backend    1/1     1            1           31s
cache      1/1     1            1           24s
frontend   1/1     1            1           37s
ubuntu@node-1:~/deploys$ kubectl get svc -n app
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
backend-svc    ClusterIP   10.233.58.10    <none>        80/TCP    40s
cache-svc      ClusterIP   10.233.27.202   <none>        80/TCP    32s
frontend-svc   ClusterIP   10.233.26.205   <none>        80/TCP    2m12s
ubuntu@node-1:~/deploys$ kubectl config set-context --current --namespace=app
Context "kubernetes-admin@cluster.local" modified.
```

Проверяем доступность:    

```
ubuntu@node-1:~/deploys$ kubectl exec frontend-9db795bbf-n4dsb -- curl backend-svc
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   140  100   140    0     0   8188      0 --:--:-- --:--:-- --:--:--  8750
WBITT Network MultiTool (with NGINX) - backend-58b6c7f7f7-n9cxf - 10.233.71.3 - HTTP: 80 , HTTPS: 443 . (Formerly praqma/network-multitool)
ubuntu@node-1:~/deploys$ kubectl exec backend-58b6c7f7f7-n9cxf -- curl cache-svc
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0WBITT Network MultiTool (with NGINX) - cache-6ddb58d8bc-x79n6 - 10.233.71.4 - HTTP: 80 , HTTPS: 443 . (Formerly praqma/network-multitool)
100   138  100   138    0     0  37438      0 --:--:-- --:--:-- --:--:-- 46000
```

Применяем дефолтную политику, запрещающую все доступа:    

```
ubuntu@node-1:~/deploys$ kubectl apply -f default-policy.yml
networkpolicy.networking.k8s.io/default-deny-ingress created
ubuntu@node-1:~/deploys$ kubectl exec frontend-9db795bbf-n4dsb -- curl backend-svc
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:01 --:--:--     0^C
ubuntu@node-1:~/deploys$ kubectl exec backend-58b6c7f7f7-n9cxf -- curl cache-svc
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:01 --:--:--     0^C
ubuntu@node-1:~/deploys$ kubectl get networkpolicies
NAME                   POD-SELECTOR   AGE
default-deny-ingress   <none>         4m55s
```

Применяем политики, указанные в условии задания:    

```
ubuntu@node-1:~/deploys$ kubectl apply -f front-back.yml
networkpolicy.networking.k8s.io/front-back created
ubuntu@node-1:~/deploys$ kubectl apply -f back-cache.yml
networkpolicy.networking.k8s.io/back-cache created
ubuntu@node-1:~/deploys$ kubectl get networkpolicies
NAME                   POD-SELECTOR   AGE
back-cache             app=cache      5s
default-deny-ingress   <none>         5m16s
front-back             app=backend    11s
ubuntu@node-1:~/deploys$ kubectl exec frontend-9db795bbf-n4dsb -- curl backend-svc
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0WBITT Network MultiTool (with NGINX) - backend-58b6c7f7f7-n9cxf - 10.233.71.3 - HTTP: 80 , HTTPS: 443 . (Formerly praqma/network-multitool)
100   140  100   140    0     0   8492      0 --:--:-- --:--:-- --:--:--  8750
ubuntu@node-1:~/deploys$ kubectl exec backend-58b6c7f7f7-n9cxf -- curl cache-svc
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0WBITT Network MultiTool (with NGINX) - cache-6ddb58d8bc-x79n6 - 10.233.71.4 - HTTP: 80 , HTTPS: 443 . (Formerly praqma/network-multitool)
100   138  100   138    0     0  11241      0 --:--:-- --:--:-- --:--:-- 11500
ubuntu@node-1:~/deploys$ kubectl exec cache-6ddb58d8bc-x79n6 -- curl frontend-svc
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:02 --:--:--     0^C
ubuntu@node-1:~/deploys$ kubectl exec backend-58b6c7f7f7-n9cxf -- curl frontend-svc
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:01 --:--:--     0^C

```

Все примененные политики успешно отработали.    

### Правила приёма работы

1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.
