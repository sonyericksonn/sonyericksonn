# Descomplicando Kubernetes Day 4

## Sumário

<!-- TOC -->

- [Descomplicando Kubernetes Day 4](#descomplicando-kubernetes-day-4)
  - [Sumário](#sumário)
- [Volumes](#volumes)
  - [Empty-Dir](#empty-dir)
  - [Persistent Volume](#persistent-volume)
- [Cron Jobs](#cron-jobs)
- [Secrets](#secrets)
- [ConfigMaps](#configmaps)
- [InitContainers](#initcontainers)
- [Criando um usuário no Kubernetes](#criando-um-usuário-no-kubernetes)
- [RBAC](#rbac)
  - [Role e ClusterRole](#role-e-clusterrole)
  - [RoleBinding e ClusterRoleBinding](#rolebinding-e-clusterrolebinding)
- [Helm](#helm)
  - [Instalando o Helm 3](#instalando-o-helm-3)
  - [Comandos Básicos do Helm 3](#comandos-básicos-do-helm-3)

<!-- TOC -->

# Volumes

## Empty-Dir

Um volume do tipo **EmptyDir** é criado sempre que um Pod é atribuído a um nó existente. Esse volume é criado inicialmente vazio, e todos os contêineres do Pod podem ler e gravar arquivos no volume.

Esse volume não é um volume com persistência de dados. Sempre que o Pod é removido de um nó, os dados no ``EmptyDir`` são excluídos permanentemente. É importante ressaltar que os dados não são excluídos em casos de falhas nos contêineres.

Vamos criar um Pod para testar esse volume:

```
vim pod-emptydir.yaml
```

Informe o seguinte conteúdo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: busybox
    name: busy
    command:
      - sleep
      - "3600"
    volumeMounts:
    - mountPath: /giropops
      name: giropops-dir
  volumes:
  - name: giropops-dir
    emptyDir: {}
```

Crie o volume a partir do manifesto.

```
kubectl create -f pod-emptydir.yaml

pod/busybox created
```

Visualize os pods:

```
kubectl get pod

NAME                      READY     STATUS    RESTARTS   AGE
busybox                   1/1       Running   0          12s
```

Pronto! Já subimos nosso pod.

Vamos listar o nome do contêiner que está dentro do pod ``busybox``:

```
kubectl get pods busybox -n default -o jsonpath='{.spec.containers[*].name}*'

busy
```

Agora vamos adicionar um arquivo dentro do path ``/giropops`` diretamente no Pod criado:

```
kubectl exec -ti busybox -c busy -- touch /giropops/funciona
```

Agora vamos listar esse diretório:

```
kubectl exec -ti busybox -c busy -- ls -l /giropops/

total 0
-rw-r--r--    1 root     root             0 Jul  7 17:37 funciona
```

Como podemos observar nosso arquivo foi criado corretamente. Vamos verificar se esse arquivo também foi criado no volume gerenciado pelo ``kubelet``. Para isso precisamos descobrir em qual nó está alocado o Pod.

```
kubectl get pod -o wide

NAME     READY     STATUS    RESTARTS   AGE  IP          NODE
busybox  1/1       Running   0          1m   10.40.0.6   elliot-02
```

Vamos acessar o shell do contêiner ``busy``, que está dentro do pod ``busybox``:

```
kubectl exec -ti busybox -c busy sh
```

Liste o conteúdo do diretório ``giropops``.

```
ls giropops
```

Agora vamos sair do Pod e procurar o nosso volume dentro do nó ``elliot-02``. Para isso acesse-o node via SSH e, em seguida, execute o comando:

```
find /var/lib/kubelet/pods/ -iname giropops-dir

/var/lib/kubelet/pods/7d33810f-8215-11e8-b889-42010a8a0002/volumes/kubernetes.io~empty-dir/giropops-dir
```

Vamos listar esse ``Path``:

```
ls /var/lib/kubelet/pods/7d33810f-8215-11e8-b889-42010a8a0002/volumes/kubernetes.io~empty-dir/giropops-dir
funciona
```

O arquivo que criamos dentro do contêiner está listado.

De volta ao nó ``elliot-01``, vamos para remover o Pod e listar novamente o diretório.

```
kubectl delete -f pod-emptydir.yaml

pod "busybox" deleted
```

Volte a acessar o nó ``elliot-02`` e veja se o volume ainda existe:

```
ls /var/lib/kubelet/pods/7d...kubernetes.io~empty-dir/giropops-dir

No such file or directory
```

Opa, recebemos a mensagem de que o diretório não pode ser encontrado, exatamente o que esperamos correto? Porque o volume do tipo **EmptyDir** não mantém os dados persistentes.

## Persistent Volume

O subsistema **PersistentVolume** fornece uma API para usuários e administradores que resume detalhes de como o armazenamento é fornecido e consumido pelos Pods. Para o melhor controle desse sistema foi introduzido dois recursos de API: ``PersistentVolume`` e ``PersistentVolumeClaim``.

Um **PersistentVolume** (PV) é um recurso no cluster, assim como um nó. Mas nesse caso é um recurso de armazenamento. O PV é uma parte do armazenamento no cluster que foi provisionado por um administrador. Os PVs tem um ciclo de vida independente de qualquer pod associado a ele. Essa API permite armazenamentos do tipo: NFS, ISCSI ou armazenamento de um provedor de nuvem específico.

Um **PersistentVolumeClaim** (PVC) é semelhante a um Pod. Os Pods consomem recursos de um nó e os PVCs consomem recursos dos PVs.

Mas o que é um PVC? Nada mais é do que uma solicitação de armazenamento criada por um usuário.

Vamos criar um ``PersistentVolume`` do tipo ``NFS``, para isso vamos instalar os pacotes necessários para criar um NFS Server no GNU/Linux.

Vamos instalar os pacotes no node ``elliot-01``.

No Debian/Ubuntu:

```
sudo apt-get install -y nfs-kernel-server
```

No CentOS/RedHat tanto no servidor quanto nos nodes o pacote será o mesmo:

```
yum install -y nfs-utils
```

Agora vamos instalar o pacote ``nfs-common`` nos demais nodes da família Debian.

```
sudo apt-get install -y nfs-common
```

Agora vamos montar um diretório no node ``elliot-01`` e dar as permissões necessárias para testar tudo isso que estamos falando:

```
sudo mkdir /opt/dados
sudo chmod 1777 /opt/dados/
```

Ainda no node ``elliot-01``, vamos adicionar esse diretório no NFS Server e fazer a ativação do mesmo.

```
sudo vim /etc/exports
```

Adicione a seguinte linha:

```
/opt/dados *(rw,sync,no_root_squash,subtree_check)
```

Aplique a configuração do NFS no node ``elliot-01``.

```
sudo exportfs -a
```

Vamos fazer o restart do serviço do NFS no node ``elliot-01``.

No Debian/Ubuntu:

```
sudo systemctl restart nfs-kernel-server
```

No CentOS/RedHat:

```
# systemctl restart nfs
```

Ainda no node ``elliot-01``, vamos criar um arquivo nesse diretório para nosso teste.

```
sudo touch /opt/dados/FUNCIONA
```

Ainda no node ``elliot-01``, vamos criar o manifesto ``yaml`` do nosso ``PersistentVolume``. Lembre-se de alterar o IP address do campo server para o IP address do node ``elliot-01``.

```
sudo vim primeiro-pv.yaml
```

Informe o seguinte conteúdo:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: primeiro-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /opt/dados
    server: 10.138.0.2
    readOnly: false
```

Agora vamos criar nosso ``PersistentVolume``:

```
kubectl create -f primeiro-pv.yaml

persistentvolume/primeiro-pv created
```

Visualizando os PVs:

```
kubectl get pv

NAME          CAPACITY   ACCESS MODES    RECLAIM POLICY   ...  AGE
primeiro-pv   1Gi        RWX             Retain           ...  22s
```

Visualizando mais detalhes do PV recém criado:

```
kubectl describe pv primeiro-pv

Name:            primeiro-pv
Labels:          <none>
Annotations:     <none>
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:
Status:          Available
Claim:
Reclaim Policy:  Retain
Access Modes:    RWX
Capacity:        1Gi
Node Affinity:   <none>
Message:
Source:
    Type:      NFS (an NFS mount that lasts the lifetime of a pod)
    Server:    10.138.0.2
    Path:      /opt/dados
    ReadOnly:  false
Events:        <none>
```

Agora precisamos criar nosso ``PersitentVolumeClaim``, assim os Pods conseguem solicitar leitura e escrita ao nosso ``PersistentVolume``.

Crie o seguinte arquivo:

```
vim primeiro-pvc.yaml
```

Informe o seguinte conteúdo:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: primeiro-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 800Mi
```

Crie o ``PersitentVolumeClaim`` a partir do manifesto.

```
kubectl create -f primeiro-pvc.yaml

persistentvolumeclaim/primeiro-pvc created
```

Vamos listar nosso ``PersistentVolume``:

```
kubectl get pv

NAME         CAPACITY  ACCESS MOD  ... CLAIM                ...  AGE
primeiro-pv  1Gi       RWX         ... default/primeiro-pvc ...  8m
```

Vamos listar nosso ``PersistentVolumeClaim``:

```
kubectl get pvc

NAME          STATUS   VOLUME       CAPACITY   ACCESS MODES ...  AGE
primeiro-pvc  Bound    primeiro-pv  1Gi        RWX          ...  3m
```

Agora que já temos um ``PersistentVolume`` e um ``PersistentVolumeClaim`` vamos criar um deployment que irá consumir esse volume:

```
vim nfs-pv.yaml
```

Informe o seguinte conteúdo:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx
  namespace: default
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      run: nginx
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        volumeMounts:
        - name: nfs-pv
          mountPath: /giropops
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      volumes:
      - name: nfs-pv
        persistentVolumeClaim:
          claimName: primeiro-pvc
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

Crie o deployment a partir do manifesto.

```
kubectl create -f nfs-pv.yaml

deployment.apps/nginx created
```

Visualize os detalhes do deployment.

```
kubectl describe deployment nginx

Name:                   nginx
Namespace:              default
CreationTimestamp:      Wed, 7 Jul 2018 22:01:49 +0000
Labels:                 run=nginx
Annotations:            deployment.kubernetes.io/revision=1
Selector:               run=nginx
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
Pod Template:
  Labels:  run=nginx
  Containers:
   nginx:
    Image:        nginx
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:
      /giropops from nfs-pv (rw)
  Volumes:
   nfs-pv:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  primeiro-pvc
    ReadOnly:   false
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-b4bd77674 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  7s    deployment-controller  Scaled up replica set nginx-b4bd77674 to 1
```

Como podemos observar detalhando nosso pod, ele foi criado com o Volume NFS utilizando o ``ClaimName`` com o valor ``primeiro-pvc``.

Vamos detalhar nosso pod para verificar se está tudo certinho.

```
kubectl get pods -o wide

NAME           READY    STATUS   RESTARTS   AGE ...  NODE
nginx-b4b...   1/1      Running  0          28s      elliot-02
```

```
kubectl describe pod nginx-b4bd77674-gwc9k

Name:               nginx-b4bd77674-gwc9k
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               elliot-02/10.138.0.3
...
    Mounts:
      /giropops from nfs-pv (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-np77m (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  nfs-pv:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  primeiro-pvc
    ReadOnly:   false
  default-token-np77m:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-np77m
    Optional:    false
...
```

Tudo certo, não? O pod realmente montou o volume ``/giropops`` utilizando o volume ``nfs``.

Agora vamos listar os arquivos dentro do path no contêiner utilizando nosso exemplo que no caso está localizado no nó `elliot-02`:

```
kubectl exec -ti nginx-b4bd77674-gwc9k -- ls /giropops/

FUNCIONA
```

Podemos ver o arquivos que lá no começo está listado no diretório.

Agora vamos criar outro arquivo utilizando o próprio contêiner com o comando ``kubectl exec`` e o parâmetro ``-- touch``.

```
kubectl exec -ti nginx-b4bd77674-gwc9k -- touch /giropops/STRIGUS

kubectl exec -ti nginx-b4bd77674-gwc9k -- ls -la  /giropops/

total 4
drwxr-xr-x. 2 root root 4096 Jul  7 23:13 .
drwxr-xr-x. 1 root root   44 Jul  7 22:53 ..
-rw-r--r--. 1 root root    0 Jul  7 22:07 FUNCIONA
-rw-r--r--. 1 root root    0 Jul  7 23:13 STRIGUS
```

Listando dentro do contêiner podemos observar que o arquivo foi criado, mas e dentro do nosso NFS Server? Vamos listar o diretório do NSF Server no ``elliot-01``.

```
ls -la /opt/dados/

-rw-r--r-- 1 root root    0 Jul  7 22:07 FUNCIONA
-rw-r--r-- 1 root root    0 Jul  7 23:13 STRIGUS
```

Viram? Nosso NFS server está funcionando corretamente, agora vamos apagar o deployment para ver o que acontece com o volume.

```
kubectl get deployment

NAME                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx               1         1         1            1           28m
```

Remova o deployment:

```
kubectl delete deployment nginx

deployment.extensions "nginx" deleted
```

Agora vamos listar o diretório no NFS Server.

```
ls -la /opt/dados/

-rw-r--r-- 1 root root    0 Jul  7 22:07 FUNCIONA
-rw-r--r-- 1 root root    0 Jul  7 23:13 STRIGUS
```

Como era de se esperar os arquivos continuam lá e não foram deletados com a exclusão do Deployment, essa é uma das várias formas de se manter arquivos persistentes para os Pods consumirem.

# Cron Jobs

Um serviço **CronJob** nada mais é do que uma linha de um arquivo crontab o mesmo arquivo de uma tabela ``cron``. Ele agenda e executa tarefas periodicamente em um determinado cronograma.

Mas para que podemos usar os **Cron Jobs****? As "Cron" são úteis para criar tarefas periódicas e recorrentes, como executar backups ou enviar e-mails.

Vamos criar um exemplo para ver como funciona, bora criar nosso manifesto:

```
vim primeiro-cron.yaml
```

Informe o seguinte conteúdo.

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: giropops-cron
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: giropops-cron
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Bem Vindo ao Descomplicando Kubernetes - LinuxTips VAIIII ;sleep 30
          restartPolicy: OnFailure
```

Nosso exemplo de ``CronJobs`` anterior imprime a hora atual e uma mensagem de de saudação a cada minuto.

Vamos criar o ``CronJob`` a partir do manifesto.

```
kubectl create -f primeiro-cron.yaml

cronjob.batch/giropops-cron created
```

Agora vamos listar e detalhar melhor nosso ``Cronjob``.

```
kubectl get cronjobs

NAME            SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
giropops-cron   */1 * * * *   False     1        13s             2m
```

Vamos visualizar os detalhes do ``Cronjob`` ``giropops-cron``.

```
kubectl describe cronjobs.batch giropops-cron

Name:                       giropops-cron
Namespace:                  default
Labels:                     <none>
Annotations:                <none>
Schedule:                   */1 * * * *
Concurrency Policy:         Allow
Suspend:                    False
Starting Deadline Seconds:  <unset>
Selector:                   <unset>
Parallelism:                <unset>
Completions:                <unset>
Pod Template:
  Labels:  <none>
  Containers:
   giropops-cron:
    Image:      busybox
    Port:       <none>
    Host Port:  <none>
    Args:
      /bin/sh
      -c
      date; echo LinuxTips VAIIII ;sleep 30
    Environment:     <none>
    Mounts:          <none>
  Volumes:           <none>
Last Schedule Time:  Wed, 22 Aug 2018 22:33:00 +0000
Active Jobs:         <none>
Events:
  Type    Reason            Age   From                Message
  ----    ------            ----  ----                -------
  Normal  SuccessfulCreate  1m    cronjob-controller  Created job giropops-cron-1534977120
  Normal  SawCompletedJob   1m    cronjob-controller  Saw completed job: giropops-cron-1534977120
  Normal  SuccessfulCreate  41s   cronjob-controller  Created job giropops-cron-1534977180
  Normal  SawCompletedJob   1s    cronjob-controller  Saw completed job: giropops-cron-1534977180
  Normal  SuccessfulDelete  1s    cronjob-controller  Deleted job giropops-cron-1534977000
```

Olha que bacana, se observar no ``Events`` do cluster o ``cron`` já está agendando e executando as tarefas.

Agora vamos ver esse ``cron`` funcionando através do comando ``kubectl get`` junto do parâmetro ``--watch`` para verificar a saída das tarefas, preste atenção que a tarefa vai ser criada em cerca de um minuto após a criação do ``CronJob``.

```
kubectl get jobs --watch

NAME                       DESIRED  SUCCESSFUL   AGE
giropops-cron-1534979640   1         1            2m
giropops-cron-1534979700   1         1            1m
```

Vamos visualizar o CronJob:

```
kubectl get cronjob giropops-cron

NAME           SCHEDULE      SUSPEND   ACTIVE    LAST SCHEDULE   AGE
giropops-cron  */1 * * * *   False     1         26s             48m
```

Como podemos observar que nosso ``cron`` está funcionando corretamente. Para visualizar a saída dos comandos executados pela tarefa vamos utilizar o comando ``logs`` do ``kubectl``.

Para isso vamos listar os pods em execução e, em seguida, pegar os logs do mesmo.

```
kubectl get pods

NAME                            READY     STATUS      RESTARTS   AGE
giropops-cron-1534979940-vcwdg  1/1       Running     0          25s
```

Vamos visualizar os logs:

```
kubectl logs giropops-cron-1534979940-vcwdg

Wed Aug 22 23:19:06 UTC 2018
LinuxTips VAIIII
```

O ``cron`` está executando corretamente as tarefas de imprimir a data e a frase que criamos no manifesto.

Se executarmos um ``kubectl get pods`` poderemos ver os Pods criados e utilizados para executar as tarefas a todo minuto.

```
kubectl get pods

NAME                             READY    STATUS      RESTARTS   AGE
giropops-cron-1534980360-cc9ng   0/1      Completed   0          2m
giropops-cron-1534980420-6czgg   0/1      Completed   0          1m
giropops-cron-1534980480-4bwcc   1/1      Running     0          4s
```

---

> OBS.: Por padrão, o Kubernetes mantém o histórico dos últimos 3 ``cron`` executados, concluídos ou com falhas.
Fonte: https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/#jobs-history-limits

---

Agora vamos deletar nosso CronJob:

```
kubectl delete cronjob giropops-cron

cronjob.batch "giropops-cron" deleted
```

# Secrets

Objetos do tipo **Secret** são normalmente utilizados para armazenar informações confidenciais, como por exemplo tokens e chaves SSH. Deixar senhas e informações confidenciais em arquivo texto não é um bom comportamento visto do olhar de segurança. Colocar essas informações em um objeto ``Secret`` permite que o administrador tenha mais controle sobre eles reduzindo assim o risco de exposição acidental.

Vamos criar nosso primeiro objeto ``Secret`` utilizando o arquivo ``secret.txt`` que vamos criar logo a seguir.

```
echo -n "giropops strigus girus" > secret.txt
```

Agora que já temos nosso arquivo ``secret.txt`` com o conteúdo ``descomplicando-k8s`` vamos criar nosso objeto ``Secret``.

```
kubectl create secret generic my-secret --from-file=secret.txt

secret/my-secret created
```

Vamos ver os detalhes desse objeto para ver o que realmente aconteceu.

```
kubectl describe secret my-secret

Name:         my-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
secret.txt:  18 bytes
```

Observe que não é possível ver o conteúdo do arquivo utilizando o ``describe``, isso é para proteger a chave de ser exposta acidentalmente.

Para verificar o conteúdo de um ``Secret`` precisamos decodificar o arquivo gerado, para fazer isso temos que verificar o manifesto do do mesmo.

```
kubectl get secret

NAME              TYPE             DATA      AGE
my-secret         Opaque           1         13m
```

```
kubectl get secret my-secret -o yaml

apiVersion: v1
data:
  secret.txt: Z2lyb3BvcHMgc3RyaWd1cyBnaXJ1cw==
kind: Secret
metadata:
  creationTimestamp: 2018-08-26T17:10:14Z
  name: my-secret
  namespace: default
  resourceVersion: "3296864"
  selfLink: /api/v1/namespaces/default/secrets/my-secret
  uid: e61d124a-a952-11e8-8723-42010a8a0002
type: Opaque
```

Agora que já temos a chave codificada basta decodificar usando ``Base64``.

```
echo 'Z2lyb3BvcHMgc3RyaWd1cyBnaXJ1cw==' | base64 --decode

giropops strigus girus
```

Tudo certo com nosso ``Secret``, agora vamos utilizar ele dentro de um Pod, para isso vamos precisar referenciar o ``Secret`` dentro do Pod utilizando volumes, vamos criar nosso manifesto.

```
vim pod-secret.yaml
```

Informe o seguinte conteúdo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-secret
  namespace: default
spec:
  containers:
  - image: busybox
    name: busy
    command:
      - sleep
      - "3600"
    volumeMounts:
    - mountPath: /tmp/giropops
      name: my-volume-secret
  volumes:
  - name: my-volume-secret
    secret:
      secretName: my-secret
```

Nesse manifesto vamos utilizar o volume ``my-volume-secret`` para montar dentro do contêiner a Secret ``my-secret`` no diretório ``/tmp/giropos``.

```
kubectl create -f pod-secret.yaml

pod/test-secret created
```

Vamos verificar se o ``Secret`` foi criado corretamente:

```
kubectl exec -ti test-secret -- ls /tmp/giropops

secret.txt
```

```
kubectl exec -ti test-secret -- cat /tmp/giropops/secret.txt

giropops strigus girus
```

Sucesso! Esse é um dos modos de colocar informações ou senha dentro de nossos Pods, mas existe um jeito ainda mais bacana utilizando os Secrets como variável de ambiente.

Vamos dar uma olhada nesse cara, primeiro vamos criar um novo objeto ``Secret`` usando chave literal com chave e valor.

```
kubectl create secret generic my-literal-secret --from-literal user=linuxtips --from-literal password=catota

secret/my-literal-secret created
```

Vamos ver os detalhes do objeto ``Secret`` ``my-literal-secret``:

```
kubectl describe secret my-literal-secret

Name:         my-literal-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  6 bytes
user:      9 bytes
```

Acabamos de criar um objeto ``Secret`` com duas chaves, um ``user`` e outra ``password``, agora vamos referenciar essa chave dentro do nosso Pod utilizando variável de ambiente, para isso vamos criar nosso novo manifesto.

```
vim pod-secret-env.yaml
```

Informe o seguinte conteúdo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: teste-secret-env
  namespace: default
spec:
  containers:
  - image: busybox
    name: busy-secret-env
    command:
      - sleep
      - "3600"
    env:
    - name: MEU_USERNAME
      valueFrom:
        secretKeyRef:
          name: my-literal-secret
          key: user
    - name: MEU_PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-literal-secret
          key: password
```

Vamos criar nosso pod.

```
kubectl create -f pod-secret-env.yaml

pod/teste-secret-env created
```

Agora vamos listar as variáveis de ambiente dentro do contêiner para verificar se nosso Secret realmente foi criado.

```
kubectl exec teste-secret-env -c busy-secret-env -it -- printenv

PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=teste-secret-env
TERM=xterm
MEU_USERNAME=linuxtips
MEU_PASSWORD=catota
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
HOME=/root
```

Viram? Agora podemos utilizar essa chave dentro do contêiner como variável de ambiente, caso alguma aplicação dentro do contêiner precise se conectar ao um banco de dados por exemplo utilizando usuário e senha, basta criar um ``secret`` com essas informações e referenciar dentro de um Pod depois é só consumir dentro do Pod como variável de ambiente ou um arquivo texto criando volumes.

# ConfigMaps

Os Objetos do tipo **ConfigMaps** são utilizados para separar arquivos de configuração do conteúdo da imagem de um contêiner, assim podemos adicionar e alterar arquivos de configuração dentro dos Pods sem buildar uma nova imagem de contêiner.

Para nosso exemplo vamos utilizar um ``ConfigMaps`` configurado com dois arquivos e  um valor literal.

Vamos criar um diretório chamado ``frutas`` e nele vamos adicionar frutas e suas características.

```
mkdir frutas

echo -n amarela > frutas/banana

echo -n vermelho > frutas/morango

echo -n verde > frutas/limao

echo -n "verde e vermelha" > frutas/melancia

echo -n kiwi > predileta
```

Crie o ``Configmap``.

```
kubectl create configmap cores-frutas --from-literal=uva=roxa --from-file=predileta --from-file=frutas/
```

Visualize o Configmap.

```
kubectl get configmap
```

Vamos criar um pod para usar o Configmap:

```
vim pod-configmap.yaml
```

Informe o seguinte conteúdo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-configmap
  namespace: default
spec:
  containers:
  - image: busybox
    name: busy-configmap
    command:
      - sleep
      - "3600"
    env:
    - name: frutas
      valueFrom:
        configMapKeyRef:
          name: cores-frutas
          key: predileta
```

Crie o pod a partir do manifesto.

```
kubectl create -f pod-configmap.yaml
```

Após a criação, execute o comando ``set`` dentro do contêiner, para listar as variáveis de ambiente e conferir se foi criada a variável de acordo com a ``key=predileta`` que definimos em nosso arquivo yaml.

Repare no final da saída do comando ``set`` a env ``frutas='kiwi'``.

```
kubectl exec -ti busybox-configmap -- sh 
/ # set
...
frutas='kiwi'
```

Vamos criar um pod utilizando utilizando mais de uma variável:

```
vim pod-configmap-env.yaml
```

Informe o seguinte conteúdo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-configmap-env
  namespace: default
spec:
  containers:
  - image: busybox
    name: busy-configmap
    command:
      - sleep
      - "3600"
    envFrom:
    - configMapRef:
        name: cores-frutas
```

Crie o pod a partir do manifesto:

```
kubectl create -f pod-configmap-env.yaml
```

Vamos entrar no contêiner e executar o comando ``set`` novamente para listar as variáveis, repare que foi criada todas as variáveis.

```
kubectl exec -ti busybox-configmap-env -- sh

/ # set
...
banana='amarela'
limao='verde'
melancia='verde e vermelha'
morango='vermelho'
predileta='kiwi'
uva='roxa'
```

Agora vamos criar um pod para usar outro Configmap, só que dessa vez utilizando volume:

```
vim pod-configmap-file.yaml
```

Informe o seguinte conteúdo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-configmap-file
  namespace: default
spec:
  containers:
  - image: busybox
    name: busy-configmap
    command:
      - sleep
      - "3600"
    volumeMounts:
    - name: meu-configmap-vol
      mountPath: /etc/frutas
  volumes:
  - name: meu-configmap-vol
    configMap:
      name: cores-frutas
```

Crie o pod a partir do manifesto.

```
kubectl create -f pod-configmap-file.yaml
```

Após a criação do pod, vamos conferir o nosso configmap como arquivos.

```
kubectl exec -ti busybox-configmap-file -- sh
/ # ls -lh /etc/frutas/
total 0      
lrwxrwxrwx    1 root     root          13 Sep 23 04:56 banana -> ..data/banana
lrwxrwxrwx    1 root     root          12 Sep 23 04:56 limao -> ..data/limao
lrwxrwxrwx    1 root     root          15 Sep 23 04:56 melancia -> ..data/melancia
lrwxrwxrwx    1 root     root          14 Sep 23 04:56 morango -> ..data/morango
lrwxrwxrwx    1 root     root          16 Sep 23 04:56 predileta -> ..data/predileta
lrwxrwxrwx    1 root     root          10 Sep 23 04:56 uva -> ..data/uva
```

# InitContainers

O objeto do tipo **InitContainers** são um ou mais contêineres que são executados antes do contêiner de um aplicativo em um Pod. Os contêineres de inicialização podem conter utilitários ou scripts de configuração não presentes em uma imagem de aplicativo.

- Os contêineres de inicialização sempre são executados até a conclusão.
- Cada contêiner init deve ser concluído com sucesso antes que o próximo comece.

Se o contêiner init de um Pod falhar, o Kubernetes reiniciará repetidamente o Pod até que o contêiner init tenha êxito. No entanto, se o Pod tiver o ``restartPolicy`` como ``Never``, o Kubernetes não reiniciará o Pod, e o contêiner principal não irá ser executado.

Crie o Pod a partir do manifesto:

```
vim nginx-initcontainer.yaml
```

Informe o seguinte conteúdo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: workdir
      mountPath: /usr/share/nginx/html
  initContainers:
  - name: install
    image: busybox
    command: ['wget','-O','/work-dir/index.html','http://linuxtips.io']
    volumeMounts:
    - name: workdir
      mountPath: "/work-dir"
  dnsPolicy: Default
  volumes:
  - name: workdir
    emptyDir: {}
```

Crie o pod a partir do manifesto.

```
kubectl create -f nginx-initcontainer.yaml

pod/init-demo created
```

Visualize o conteúdo de um arquivo dentro de um contêiner do Pod.

```
kubectl exec -ti init-demo -- cat /usr/share/nginx/html/index.html
```

Vamos ver os detalhes do Pod:

```
kubectl describe pod init-demo

Name:               init-demo
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               k8s3/172.31.28.13
Start Time:         Sun, 11 Nov 2018 13:10:02 +0000
Labels:             <none>
Annotations:        <none>
Status:             Running
IP:                 10.44.0.13
Init Containers:
  install:
    Container ID:  docker://d86ec7ce801819a2073b96098055407dec5564a678c2548dd445a613314bac8e
    Image:         busybox
    Image ID:      docker-pullable://busybox@sha256:2a03a6059f21e150ae84b0973863609494aad70f0a80eaeb64bddd8d92465812
    Port:          <none>
    Host Port:     <none>
    Command:
      wget
      -O
      /work-dir/index.html
      http://kubernetes.io
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Sun, 11 Nov 2018 13:10:03 +0000
      Finished:     Sun, 11 Nov 2018 13:10:03 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-x5g8c (ro)
      /work-dir from workdir (rw)
Containers:
  nginx:
    Container ID:   docker://1ec37249f98ececdfb5f80c910142c5e6edbc9373e34cd194fbd3955725da334
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:d59a1aa7866258751a261bae525a1842c7ff0662d4f34a355d5f36826abc0341
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sun, 11 Nov 2018 13:10:04 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /usr/share/nginx/html from workdir (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-x5g8c (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  workdir:
    Type:    EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
  default-token-x5g8c:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-x5g8c
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  3m1s   default-scheduler  Successfully assigned default/init-demo to k8s3
  Normal  Pulling    3m     kubelet, k8s3      pulling image "busybox"
  Normal  Pulled     3m     kubelet, k8s3      Successfully pulled image "busybox"
  Normal  Created    3m     kubelet, k8s3      Created container
  Normal  Started    3m     kubelet, k8s3      Started container
  Normal  Pulling    2m59s  kubelet, k8s3      pulling image "nginx"
  Normal  Pulled     2m59s  kubelet, k8s3      Successfully pulled image "nginx"
  Normal  Created    2m59s  kubelet, k8s3      Created container
  Normal  Started    2m59s  kubelet, k8s3      Started container
```

Coletando os logs do contêiner init:

```
kubectl logs init-demo -c install

Connecting to linuxtips.io (23.236.62.147:80)
Connecting to www.linuxtips.io (35.247.254.172:443)
wget: note: TLS certificate validation not implemented
saving to '/work-dir/index.html'
index.html           100% |********************************|  765k  0:00:00 ETA
'/work-dir/index.html' saved
```

E por último vamos remover o Pod a partir do manifesto:

```
kubectl delete -f nginx-initcontainer.yaml

pod/init-demo deleted
```

# Criando um usuário no Kubernetes

Para criar um usuário no Kubernetes, vamos precisar gerar um CSR (*Certificate Signing Request*) para o usuário. O usuário que vamos utilizar como exemplo é o ``linuxtips``.

Comando para gerar o CSR:

```
openssl req -new -newkey rsa:4096 -nodes -keyout linuxtips.key -out linuxtips.csr -subj "/CN=linuxtips"
```

Agora vamos fazer o request do CSR no cluster:

```
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: linuxtips-csr
spec:
  groups:
  - system:authenticated
  request: $(cat linuxtips.csr | base64 | tr -d '\n')
  usages:
  - client auth
EOF
```

Para ver os CSR criados, utilize o seguinte comando:

```
kubectl get csr
```

O CSR deverá estar com o status ``Pending``, vamos aprová-lo

```
kubectl certificate approve linuxtips-csr
```

Agora o certificado foi assinado pela CA (*Certificate Authority*) do cluster, para pegar o certificado assinado vamos usar o seguinte comando:

```
kubectl get csr linuxtips-csr -o jsonpath='{.status.certificate}' | base64 --decode > linuxtips.crt
```

Será necessário para a configuração do ``kubeconfig`` o arquivo referente a CA do cluster, para obtê-lá vamos extrai-lá do ``kubeconf`` atual que estamos utilizando:

```
kubectl config view -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' --raw | base64 --decode - > ca.crt
```

Feito isso vamos montar nosso ``kubeconfig`` para o novo usuário:

Vamos pegar as informações de IP cluster:

```
kubectl config set-cluster $(kubectl config view -o jsonpath='{.clusters[0].name}') --server=$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}') --certificate-authority=ca.crt --kubeconfig=linuxtips-config --embed-certs
```

Agora setando as confs de ``user`` e ``key``:

```
kubectl config set-credentials linuxtips --client-certificate=linuxtips.crt --client-key=linuxtips.key --embed-certs --kubeconfig=linuxtips-config
```

Agora vamos definir o ``context linuxtips`` e, em seguida, vamos utilizá-lo:

```
kubectl config set-context linuxtips --cluster=$(kubectl config view -o jsonpath='{.clusters[0].name}')  --user=linuxtips --kubeconfig=linuxtips-config
```

```
kubectl config use-context linuxtips --kubeconfig=linuxtips-config
```

Vamos ver um teste:

```
kubectl version --kubeconfig=linuxtips-config
```

Pronto! Agora só associar um ``role`` com as permissões desejadas para o usuário.


# RBAC

O controle de acesso baseado em funções (*Role-based Access Control* - RBAC) é um método para fazer o controle de acesso aos recursos do Kubernetes com base nas funções dos administradores individuais em sua organização.

A autorização RBAC usa o ``rbac.authorization.k8s.io`` para conduzir as decisões de autorização, permitindo que você configure políticas dinamicamente por meio da API Kubernetes.

## Role e ClusterRole

Um RBAC ``Role`` ou ``ClusterRole`` contém regras que representam um conjunto de permissões. As permissões são puramente aditivas (não há regras de "negação").

- Uma ``Role`` sempre define permissões em um determinado namespace, ao criar uma função, você deve especificar o namespace ao qual ela pertence.
- O ``ClusterRole``, por outro lado, é um recurso sem namespaces.

Você pode usar um ``ClusterRole`` para:

* Definir permissões em recursos com namespace e ser concedido dentro de namespaces individuais;
* Definir permissões em recursos com namespaces e ser concedido em todos os namespaces;
* Definir permissões em recursos com escopo de cluster.

Se você quiser definir uma função em um namespace, use uma ``Role``, caso queria definir uma função em todo o cluster, use um ``ClusterRole``. :)

## RoleBinding e ClusterRoleBinding

Uma ``RoleBinding`` concede as permissões definidas em uma função a um usuário ou conjunto de usuários. Ele contém uma lista de assuntos (usuários, grupos ou contas de serviço) e uma referência à função que está sendo concedida. Um ``RoleBinding`` concede permissões dentro de um namespace específico, enquanto um ``ClusterRoleBinding`` concede esse acesso a todo o cluster.

Um RoleBinding pode fazer referência a qualquer papel no mesmo namespace. Como alternativa, um RoleBinding pode fazer referência a um ClusterRole e vincular esse ClusterRole ao namespace do RoleBinding. Se você deseja vincular um ClusterRole a todos os namespaces em seu cluster, use um ClusterRoleBinding.

Com o ``ClusterRoleBinding`` você pode associar uma conta de serviço a uma determinada ClusterRole.

Primeiro vamos exibir as ClusterRoles:

```
kubectl get clusterrole

NAME                                                                   CREATED AT
admin                                                                  2020-09-20T04:35:27Z
calico-kube-controllers                                                2020-09-20T04:35:34Z
calico-node                                                            2020-09-20T04:35:34Z
cluster-admin                                                          2020-09-20T04:35:27Z
edit                                                                   2020-09-20T04:35:27Z
kubeadm:get-nodes                                                      2020-09-20T04:35:30Z
system:aggregate-to-admin                                              2020-09-20T04:35:28Z
system:aggregate-to-edit                                               2020-09-20T04:35:28Z
system:aggregate-to-view                                               2020-09-20T04:35:28Z
system:auth-delegator                                                  2020-09-20T04:35:28Z
system:basic-user                                                      2020-09-20T04:35:27Z
system:certificates.k8s.io:certificatesigningrequests:nodeclient       2020-09-20T04:35:28Z
system:certificates.k8s.io:certificatesigningrequests:selfnodeclient   2020-09-20T04:35:28Z
system:certificates.k8s.io:kube-apiserver-client-approver              2020-09-20T04:35:28Z
system:certificates.k8s.io:kube-apiserver-client-kubelet-approver      2020-09-20T04:35:28Z
system:certificates.k8s.io:kubelet-serving-approver                    2020-09-20T04:35:28Z
system:certificates.k8s.io:legacy-unknown-approver                     2020-09-20T04:35:28Z
system:controller:attachdetach-controller                              2020-09-20T04:35:28Z
system:controller:certificate-controller                               2020-09-20T04:35:28Z
system:controller:clusterrole-aggregation-controller                   2020-09-20T04:35:28Z
system:controller:cronjob-controller                                   2020-09-20T04:35:28Z
system:controller:daemon-set-controller                                2020-09-20T04:35:28Z
system:controller:deployment-controller                                2020-09-20T04:35:28Z
system:controller:disruption-controller                                2020-09-20T04:35:28Z
system:controller:endpoint-controller                                  2020-09-20T04:35:28Z
system:controller:endpointslice-controller                             2020-09-20T04:35:28Z
system:controller:endpointslicemirroring-controller                    2020-09-20T04:35:28Z
system:controller:expand-controller                                    2020-09-20T04:35:28Z
system:controller:generic-garbage-collector                            2020-09-20T04:35:28Z
system:controller:horizontal-pod-autoscaler                            2020-09-20T04:35:28Z
system:controller:job-controller                                       2020-09-20T04:35:28Z
system:controller:namespace-controller                                 2020-09-20T04:35:28Z
system:controller:node-controller                                      2020-09-20T04:35:28Z
system:controller:persistent-volume-binder                             2020-09-20T04:35:28Z
system:controller:pod-garbage-collector                                2020-09-20T04:35:28Z
system:controller:pv-protection-controller                             2020-09-20T04:35:28Z
system:controller:pvc-protection-controller                            2020-09-20T04:35:28Z
system:controller:replicaset-controller                                2020-09-20T04:35:28Z
system:controller:replication-controller                               2020-09-20T04:35:28Z
system:controller:resourcequota-controller                             2020-09-20T04:35:28Z
system:controller:route-controller                                     2020-09-20T04:35:28Z
system:controller:service-account-controller                           2020-09-20T04:35:28Z
system:controller:service-controller                                   2020-09-20T04:35:28Z
system:controller:statefulset-controller                               2020-09-20T04:35:28Z
system:controller:ttl-controller                                       2020-09-20T04:35:28Z
system:coredns                                                         2020-09-20T04:35:30Z
system:discovery                                                       2020-09-20T04:35:27Z
system:heapster                                                        2020-09-20T04:35:28Z
system:kube-aggregator                                                 2020-09-20T04:35:28Z
system:kube-controller-manager                                         2020-09-20T04:35:28Z
system:kube-dns                                                        2020-09-20T04:35:28Z
system:kube-scheduler                                                  2020-09-20T04:35:28Z
system:kubelet-api-admin                                               2020-09-20T04:35:28Z
system:node                                                            2020-09-20T04:35:28Z
system:node-bootstrapper                                               2020-09-20T04:35:28Z
system:node-problem-detector                                           2020-09-20T04:35:28Z
system:node-proxier                                                    2020-09-20T04:35:28Z
system:persistent-volume-provisioner                                   2020-09-20T04:35:28Z
system:public-info-viewer                                              2020-09-20T04:35:27Z
system:volume-scheduler                                                2020-09-20T04:35:28Z
view                                                                   2020-09-20T04:35:28Z
```

Lembra que falamos anteriormente que para associar uma ``ClusterRole``, precisamos criar uma ``ClusterRoleBinding``?

Para ver a função de cada uma delas, você pode executar o ``describe``, como fizemos para o **cluster-admin**, repare em ``Resources`` que tem o ``*.*``, ou seja, a ``ServiceAccount`` que pertence a este grupo, pode fazer tudo dentro do cluster.

```
kubectl describe clusterrole cluster-admin

Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *.*        []                 []              [*]
             [*]                []              [*]
```

Vamos criar uma ``ServiceAccount`` na unha para ser administrador do nosso cluster:

```
kubectl create serviceaccount jeferson
```

Conferindo se a ``ServiceAccount`` foi criada:

```
kubectl get serviceaccounts

NAME      SECRETS   AGE
default   1         10d
jeferson  1         10s
```

Visualizando detalhes da ``ServiceAccount``:

```
kubectl describe serviceaccounts jeferson

Name:                jeferson
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   jeferson-token-h8dz6
Tokens:              jeferson-token-h8dz6
Events:              <none>
```

Crie uma ``ClusterRoleBinding`` e associe a ``ServiceAccount`` **jeferson** e com a função ``ClusterRole`` **cluster-admin**:

```
kubectl create clusterrolebinding toskeria --serviceaccount=default:jeferson --clusterrole=cluster-admin
```

Exibindo a ``ClusterRoleBinding`` que acabamos de criar:

```
kubectl get clusterrolebindings.rbac.authorization.k8s.io toskeria

NAME       ROLE                        AGE
toskeria   ClusterRole/cluster-admin   118m
```

Vamos mandar um ``describe`` do ``ClusterRoleBinding`` **toskeira**.

```
kubectl describe clusterrolebindings.rbac.authorization.k8s.io toskeria

Name:         toskeria
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  cluster-admin
Subjects:
  Kind            Name      Namespace
  ----            ----      ---------
  ServiceAccount  jeferson  default
```

Agora iremos criar um ``ServiceAccount`` **admin-user** a partir do manifesto ``admin-user.yaml``.

Crie o arquivo:

```
vim admin-user.yaml
```

Informe o seguinte conteúdo:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
```

Crie a ``ServiceAccount`` partir do manifesto.

```
kubectl create -f admin-user.yaml
```

Checando se a ``ServiceAccount`` foi criada:

```
kubectl get serviceaccounts --namespace=kube-system admin-user

NAME         SECRETS   AGE
admin-user   1         10s
```

Agora iremos associar o **admin-user** ao **cluster-admin** também a partir do manifesto ``admin-cluster-role-binding.yaml``.

Crie o arquivo:

```
vim admin-cluster-role-binding.yaml
```

Informe o seguinte conteúdo:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

Crie o ``ClusterRoleBinding`` partir do manifesto.

```
kubectl create -f admin-cluster-role-binding.yaml

clusterrolebinding.rbac.authorization.k8s.io/admin-user created
```

Cheque se o ``ClusterRoleBinding`` foi criado:

```
kubectl get clusterrolebindings.rbac.authorization.k8s.io admin-user

NAME         ROLE                        AGE
admin-user   ClusterRole/cluster-admin   21m
```

Pronto! Agora o usuário **admin-user** foi associado ao ``ClusterRoleBinding`` **admin-user** com a função **cluster-admin**.

Lembrando que toda vez que nos referirmos ao ``ServiceAccount``, estamos referindo à uma conta de usuário.

# Helm

O **Helm** é o gerenciador de pacotes do Kubernetes. Os pacotes gerenciados pelo Helm, são chamados de **charts**, que basicamente são formados por um conjunto de manifestos Kubernetes no formato YAML e alguns templates que ajudam a manter variáveis dinâmicas de acordo com o ambiente. O Helm ajuda você a definir, instalar e atualizar até o aplicativo Kubernetes mais complexo.

Os Helm charts são fáceis de criar, versionar, compartilhar e publicar.

O Helm é um projeto graduado no CNCF e é mantido pela comunidade, assim como o Kubernetes.

Para obter mais informações sobre o Helm, acesse os seguintes links:

* https://helm.sh
* https://helm.sh/docs/intro/quickstart
* https://www.youtube.com/watch?v=Zzwq9FmZdsU&t=2s
* https://helm.sh/docs/topics/architecture

---

> OBS.: É bom utilizar o Helm3 ao invés de Helm2.
>
> Fonte: https://helm.sh/docs/topics/v2_v3_migration

---

## Instalando o Helm 3

Execute o seguinte comando para instalar o Helm3 no node ``elliot-01``:

```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash -
```

Visualize a versão do Helm:

```
helm version
```

## Comandos Básicos do Helm 3

Vamos adicionar os repositórios oficiais de Helm charts estáveis do Prometheus e do Grafana:

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
```

Vamos listar os repositórios Helm adicionados:

```
helm repo list

NAME    URL
prometheus-community    https://prometheus-community.github.io/helm-charts
grafana                 https://grafana.github.io/helm-charts             
```

---

> OBS.: Para remover um repositório Helm, execute o comando: `helm repo remove NOME_REPOSITORIO`.

---

Vamos obter a lista atualizada de Helm charts disponíveis para instalação utilizando todos os repositórios Helm adicionados. Seria o mesmo que executar o comando ``apt-get update``.

```
helm repo update
```

Por enquanto só temos um repositório adicionado. Se tivéssemos adicionados mais, esse comando nos ajudaria muito.

Agora vamos listar quais os charts estão disponíveis para ser instalados:

```
helm search repo
```

Temos ainda a opção de listar os charts disponíveis a partir do Helm Hub (tipo o Docker Hub):

```
helm search hub
```

---

> OBS.: O comando ``helm search repo`` pode ser utilizado para listar os charts em todos os repositórios adicionados.

---

Vamos instalar o Prometheus utilizando o Helm. Mas antes vamos visualizar qual a versão do chart disponível para instalação:

```
helm search repo prometheus-community
                                 
NAME                                                    CHART VERSION   APP VERSION     DESCRIPTION                                       
prometheus-community/prometheus                         11.16.2         2.21.0          Prometheus is a monitoring system and time seri...
prometheus-community/prometheus-adapter                 2.7.0           v0.7.0          A Helm chart for k8s prometheus adapter           
prometheus-community/prometheus-blackbox-exporter       4.7.0           0.17.0          Prometheus Blackbox Exporter                      
...
```

* A coluna **NAME** mostra o nome do repositório e o chart.
* A coluna **CHART VERSION** mostra apenas a versão mais recente do chart disponível para instalação.
* A coluna **APP VERSION** mostra apenas a versão mais recente da aplicação a ser instalada pelo chart. Mas nem todo o time de desenvolvimento mantém a versão da aplicação atualizada no campo *APP VERSION*. Eles fazem isso para evitar gerar uma nova versão do chart só porque a aplicação mudou, sem haver mudanças na estrutura do chart. Dependendo de como o chart é desenvolvido, a versão da aplicação é alterada apenas no manifesto ``values.yaml`` que cita qual a imagem Docker que será instalada pelo chart.

---

> OBS.: O comando ``helm search repo prometheus -l`` exibe todas as versões do chart ``prometheus`` disponíveis para instalação.

---

Agora sim, vamos finalmente instalar o Prometheus no namespace ``default``:

```
helm install meu-prometheus --version=11.16.2 prometheus-community/prometheus

NAME: meu-prometheus
LAST DEPLOYED: Sun Oct 25 15:31:07 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
meu-prometheus-2-server.default.svc.cluster.local


Get the Prometheus server URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace default -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace default port-forward $POD_NAME 9090


The Prometheus alertmanager can be accessed via port 80 on the following DNS name from within your cluster:
meu-prometheus-2-alertmanager.default.svc.cluster.local


Get the Alertmanager URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace default -l "app=prometheus,component=alertmanager" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace default port-forward $POD_NAME 9093
#################################################################################
######   WARNING: Pod Security Policy has been moved to a global property.  #####
######            use .Values.podSecurityPolicy.enabled with pod-based      #####
######            annotations                                               #####
######            (e.g. .Values.nodeExporter.podSecurityPolicy.annotations) #####
#################################################################################


The Prometheus PushGateway can be accessed via port 9091 on the following DNS name from within your cluster:
meu-prometheus-2-pushgateway.default.svc.cluster.local


Get the PushGateway URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace default -l "app=prometheus,component=pushgateway" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace default port-forward $POD_NAME 9091

For more information on running Prometheus, visit:
https://prometheus.io/
```

---

> OBS.: Se a opção ``-n NOME_NAMESPACE`` for utilizada, a aplicação será instalada no namespace específico. O Helm na versão 3 não cria o namespace. É necessário criá-lo antes e já vimos como fazer isso no dia 2.

---

Liste as aplicações instaladas com o Helm no namespace ``default``:

```
helm list

NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
meu-prometheus  default         1               2020-10-25 12:41:05.370061181 -0300 -03 deployed        prometheus-11.16.2      2.21.0     
```

Simples como voar, não é mesmo?

Mas quando vamos verificar o status dos Pods verá que eles estarão com status de Pending. Por que será?

```
kubectl get pods

NAME                                                 READY   STATUS      RESTARTS   AGE
meu-prometheus-alertmanager-8657c8b9b8-kx4lw         0/2     Pending     0          7m51s
meu-prometheus-kube-state-metrics-6864cf55db-jm596   1/1     Running     0          7m51s
meu-prometheus-node-exporter-5bcr8                   1/1     Running     0          7m51s
meu-prometheus-node-exporter-hqpdx                   1/1     Running     0          7m51s
meu-prometheus-node-exporter-qbzpd                   1/1     Running     0          7m51s
meu-prometheus-pushgateway-667bdbcc56-6sbt9          1/1     Running     0          7m51s
meu-prometheus-server-5bc59849fd-b29q4               0/2     Pending     0          7m51s
```

Executando um describe do Pod ``meu-prometheus-server`` e verá que ele está pedindo um PVC.

```
Events:
  Type     Reason            Age                  From               Message
  ----     ------            ----                 ----               -------
  Warning  FailedScheduling  57s (x4 over 2m17s)  default-scheduler  running "VolumeBinding" filter plugin for pod "meu-prometheus-server-5bc59849fd-b29q4": pod has unbound immediate PersistentVolumeClaims
```

Problema detectado. Ele não está conseguindo montar pois não existe um ``PersistentVolumeClaims`` para ele.

Vamos preparar os PVCs para o ``prometheus`` e ``alertmanager`` criando novos diretórios no nosso querido NFS no ``elliot-01``:

```
sudo mkdir -p /opt/{alertmanager,prometheus}

sudo chmod -R 777 /opt/alertmanager/

sudo chmod -R 777 /opt/prometheus/
```

Adicione as linhas para mapear os diretório para dentro do NFS:

```
sudo vim /etc/exportfs

    /opt/prometheus *(rw,sync,subtree_check,no_root_squash)
    /opt/alertmanager *(rw,sync,subtree_check,no_root_squash)
```

Feito isso atualize o mapeamento do NFS:

```
exportfs -ar
```

Valide rodando o comando exportfs -v

```
exportfs -v

/opt/prometheus
		<world>(rw,wdelay,no_root_squash,sec=sys,rw,secure,no_root_squash,no_all_squash)
/opt/alertmanager
		<world>(rw,wdelay,no_root_squash,sec=sys,rw,secure,no_root_squash,no_all_squash)
```

Agora para finalizar vamos fazer a criação do PV e PVC para que os nossos Pods possam montar o volume dentro deles executando o ``yaml`` a seguir:

```
vim volume-prometheus.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: meu-prometheus-server
spec:
  capacity:
    storage: 8Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /opt/prometheus
    server: 10.138.0.2
    readOnly: false
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: meu-prometheus-alertmanager
spec:
  capacity:
    storage: 8Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /opt/alertmanager
    server: 10.138.0.2
    readOnly: false

```

Agora crie os persistents volumes:

```
kubectl create -f volume-prometheus.yaml
```

Valide se os PV e PVCs foram criados corretamente:

```
kubectl get pv,pvc

NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM   STORAGECLASS   REASON   AGE
persistentvolume/meu-prometheus-alertmanager   8Gi  RWO Retain Bound default/meu-prometheus-alertmanager 12m
persistentvolume/meu-prometheus-server   8Gi  RWO Retain Bound default/meu-prometheus-server 12m

NAME    STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/meu-prometheus-alertmanager   Bound    meu-prometheus-alertmanager   8Gi RWO 12m
persistentvolumeclaim/meu-prometheus-server     Bound    meu-prometheus-server 8Gi  RWO  12m
```

Agora valide se os Pods subiram:

```
kubectl get pods

meu-prometheus-alertmanager-8657c8b9b8-kx4lw         2/2     Running     0          17m
meu-prometheus-kube-state-metrics-6864cf55db-zlbwg   1/1     Running     0          17m
meu-prometheus-node-exporter-692m6                   1/1     Running     0          17m
meu-prometheus-node-exporter-qq8gf                   1/1     Running     0          17m
meu-prometheus-pushgateway-667bdbcc56-9m4mr          1/1     Running     0          17m
meu-prometheus-server-5bc59849fd-b29q490             2/2     Running     0          17m
```

Top da Balada! Veja os detalhes do pod ``meu-prometheus-server-5bc59849fd-b29q490``:

```
kubect describe pod meu-prometheus-server-5bc59849fd-b29q490

Name:         meu-prometheus-server-5bc59849fd-b29q4
Namespace:    default
Priority:     0
Node:         kube-worker1/10.128.0.12
Start Time:   Sun, 07 Jun 2020 14:39:44 +0000
Labels:       app=prometheus
              chart=prometheus-11.4.0
              component=server
              heritage=Helm
              pod-template-hash=5bc59849fd
              release=meu-prometheus
Annotations:  <none>
Status:       Running
IP:           10.40.0.2
IPs:
  IP:           10.40.0.2
Controlled By:  ReplicaSet/meu-prometheus-server-5bc59849fd
Containers:
  prometheus-server-configmap-reload:
    Container ID:  docker://b9f7255883104fc149ee4c74163357b9d379b878d19a81fd0cc22dff177fa7d4
    Image:         jimmidyson/configmap-reload:v0.3.0
    Image ID:      docker-pullable://jimmidyson/configmap-reload@sha256:d107c7a235c266273b1c3502a391fec374430e5625539403d0de797fa9c556a2
    Port:          <none>
    Host Port:     <none>
    Args:
      --volume-dir=/etc/config
      --webhook-url=http://127.0.0.1:9090/-/reload
    State:          Running
      Started:      Sun, 07 Jun 2020 14:39:50 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /etc/config from config-volume (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from meu-prometheus-server-token-g5w82 (ro)
  prometheus-server:
    Container ID:  docker://96e0e07c77c3412a68d8cce6da103265f3c8783fb6b31f1813f010c4d20728dd
    Image:         prom/prometheus:v2.18.1
    Image ID:      docker-pullable://prom/prometheus@sha256:5880ec936055fad18ccee798d2a63f64ed85bd28e8e0af17c6923a090b686c3d
    Port:          9090/TCP
    Host Port:     0/TCP
    Args:
      --storage.tsdb.retention.time=15d
      --config.file=/etc/config/prometheus.yml
      --storage.tsdb.path=/data
      --web.console.libraries=/etc/prometheus/console_libraries
      --web.console.templates=/etc/prometheus/consoles
      --web.enable-lifecycle
    State:          Running
      Started:      Sun, 07 Jun 2020 14:39:56 +0000
    Ready:          True
    Restart Count:  0
    Liveness:       http-get http://:9090/-/healthy delay=30s timeout=30s period=10s #success=1 #failure=3
    Readiness:      http-get http://:9090/-/ready delay=30s timeout=30s period=10s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /data from storage-volume (rw)
      /etc/config from config-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from meu-prometheus-server-token-g5w82 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  config-volume:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      meu-prometheus-server
    Optional:  false
  storage-volume:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  <unset>
  meu-prometheus-server-token-g5w82:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  meu-prometheus-server-token-g5w82
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age   From                   Message
  ----    ------     ----  ----                   -------
  Normal  Scheduled  12m   default-scheduler      Successfully assigned default/meu-prometheus-server-5bc59849fd-b29q4 to kube-worker1
  Normal  Pulling    12m   kubelet, kube-worker1  Pulling image "jimmidyson/configmap-reload:v0.3.0"
  Normal  Pulled     12m   kubelet, kube-worker1  Successfully pulled image "jimmidyson/configmap-reload:v0.3.0"
  Normal  Created    12m   kubelet, kube-worker1  Created container prometheus-server-configmap-reload
  Normal  Started    12m   kubelet, kube-worker1  Started container prometheus-server-configmap-reload
  Normal  Pulling    12m   kubelet, kube-worker1  Pulling image "prom/prometheus:v2.18.1"
  Normal  Pulled     12m   kubelet, kube-worker1  Successfully pulled image "prom/prometheus:v2.18.1"
  Normal  Created    12m   kubelet, kube-worker1  Created container prometheus-server
  Normal  Started    12m   kubelet, kube-worker1  Started container prometheus-server
```

Podemos ver nas linhas ``Liveness`` e ``Readness`` que o Prometheus está sendo executado na porta 9090/TCP.

---

> OBS.: Para quem está utilizando o kubectl e o helm instalado na sua máquina, pode criar um redirecionamento entre a porta 9090/TCP do pod e a porta 9091/TCP da sua máquina:

```
kubectl port-forward meu-prometheus-server-5bc59849fd-b29q4 --namespace default 9091:9090
```

Agora acesse o navegador no endereço http://localhost:9091. Mágico não é mesmo?

O comando ``kubectl port-forward`` cria um redirecionamento do tráfego 9091/TCP da sua máquina para a porta 9090 do pod que está no seu cluster. Saiba mais em:

https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/#forward-a-local-port-to-a-port-on-the-pod

---

Vamos visualizar os deployments:

```
kubectl get deployment

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
meu-prometheus-alertmanager         1/1     1            1           22m
meu-prometheus-kube-state-metrics   1/1     1            1           22m
meu-prometheus-pushgateway          1/1     1            1           22m
meu-prometheus-server               1/1     1            1           22m
```

Percebeu? Foi o Helm quem criou os deployments ao instalar a aplicação ``meu-prometheus``.

Visualize os replicaSet:

```
kubectl get replicaset

NAME                                           DESIRED   CURRENT   READY   AGE
meu-prometheus-alertmanager-8657c8b9b8         1         1         1       25m
meu-prometheus-kube-state-metrics-6864cf55db   1         1         1       25m
meu-prometheus-pushgateway-667bdbcc56          1         1         1       25m
meu-prometheus-server-5bc59849fd               1         1         1       25m
```

Visualize o services:

```
kubectl get services

NAME                                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes                          ClusterIP   10.96.0.1        <none>        443/TCP    21d
meu-prometheus-alertmanager         ClusterIP   10.110.143.219   <none>        80/TCP     26m
meu-prometheus-kube-state-metrics   ClusterIP   10.103.247.154   <none>        8080/TCP   26m
meu-prometheus-node-exporter        ClusterIP   None             <none>        9100/TCP   26m
meu-prometheus-pushgateway          ClusterIP   10.102.246.46    <none>        9091/TCP   26m
meu-prometheus-server               ClusterIP   10.110.177.99    <none>        80/TCP     26m
```

O Helm também criou esses objetos no cluster ao instalar a aplicação ``meu-prometheus``.

Vamos instalar o Grafana usando o Helm. Mas antes vamos visualizar qual a versão do chart mais recente:

```
helm search repo grafana

NAME            CHART VERSION   APP VERSION     DESCRIPTION
stable/grafana  5.1.4           7.0.3           The leading tool for querying and visualizing
```

Agora sim, vamos instalar a aplicação ``meu-grafana``:

```
helm install meu-grafana --version=5.8.12 grafana/grafana                                                                                                           

NAME: meu-grafana
LAST DEPLOYED: Sun Oct 25 15:29:12 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:

   kubectl get secret --namespace default meu-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:

   meu-grafana.default.svc.cluster.local

   Get the Grafana URL to visit by running these commands in the same shell:

     export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=meu-grafana" -o jsonpath="{.items[0].metadata.name}")
     kubectl --namespace default port-forward $POD_NAME 3000

3. Login with the password from step 1 and the username: admin
#################################################################################
######   WARNING: Persistence is disabled!!! You will lose your data when   #####
######            the Grafana pod is terminated.                            #####
#################################################################################
```

Observe que logo após a instalação do Grafana, tal como ocorreu com o Prometheus, são exibidas algumas instruções sobre como acessar a aplicação.

Essas instruções podem ser visualizadas a qualquer momento usando os comandos:

```
helm status meu-grafana
helm status meu-prometheus
```

Vamos listar as aplicações instaladas pelo Helm em todos os namespaces:

```
helm list --all

NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
meu-grafana     default         1               2020-10-25 15:29:12.373064176 -0300 -03 deployed        grafana-5.8.12          7.2.1      
meu-prometheus  default         1               2020-10-25 14:44:57.409281829 -0300 -03 deployed        prometheus-11.16.2      2.21.0     
```

Observe que a coluna **REVISION** mostra a revisão para cada aplicação instalada.

Vamos remover a aplicação ``meu-prometheus``:

```
helm uninstall meu-prometheus --keep-history
```

Liste os pods:

```
kubectl get pods -A
```

O Prometheus foi removido, certo?

Vamos listar novamente as aplicações instaladas pelo Helm em todos os namespaces:

```
helm list --all

NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
meu-grafana     default         1               2020-10-25 15:29:12.373064176 -0300 -03 deployed        grafana-5.8.12  7.2.1      
```

Agora vamos fazer o rollback da remoção da aplicação ``meu-prometheus``, informando a **revision 1**:

```
helm rollback meu-prometheus 1

Rollback was a success! Happy Helming!
```

Liste as aplicações instaladas pelo Helm em todos os namespaces:

```
helm list --all

NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
meu-grafana     default         1               2020-10-25 15:29:12.373064176 -0300 -03 deployed        grafana-5.8.12          7.2.1      
meu-prometheus  default         2               2020-10-25 15:38:24.320598236 -0300 -03 deployed        prometheus-11.16.2      2.21.0     
```

Olha o Prometheus de volta! A **revision** do Prometheus foi incrementada para **2**.

A revision sempre é incrementada a cada alteração no deploy de uma aplicação. Essa mesma estratégia de rollback pode ser aplicada quando estiver fazendo uma atualização de uma aplicação.

Vamos visualizar o histórico de mudanças da aplicação ``meu-prometheus``:

```
helm history meu-prometheus

REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION            
1               Sun Oct 25 15:37:29 2020        uninstalled     prometheus-11.16.2      2.21.0          Uninstallation complete
2               Sun Oct 25 15:38:24 2020        deployed        prometheus-11.16.2      2.21.0          Rollback to 1          
```

Se a aplicação for removida sem a opção `--keep-history`, o histórico será perdido e não será possível fazer rollback.

Para testar isso, remova as aplicações ``meu-prometheus`` e ``meu-grafana`` com os seguintes comandos:

```
helm uninstall meu-grafana
helm uninstall meu-prometheus
```

Liste as aplicações instaladas em todos os namespaces:

```
helm list --all
```
