- [Introdução](README.md)
<!-- TOC -->
- [Descomplicando Kubernetes Day 1](day-1/DescomplicandoKubernetes-Day1.md#descomplicando-kubernetes-day-1)
  - [Sumário](day-1/DescomplicandoKubernetes-Day1.md#sumário)
- [O quê preciso saber antes de começar?](day-1/DescomplicandoKubernetes-Day1.md#o-quê-preciso-saber-antes-de-começar)
  - [Qual distro GNU/Linux devo usar?](day-1/DescomplicandoKubernetes-Day1.md#qual-distro-gnulinux-devo-usar)
  - [Alguns sites que devemos visitar](day-1/DescomplicandoKubernetes-Day1.md#alguns-sites-que-devemos-visitar)
  - [E o k8s?](day-1/DescomplicandoKubernetes-Day1.md#e-o-k8s)
  - [Arquitetura do k8s](day-1/DescomplicandoKubernetes-Day1.md#arquitetura-do-k8s)
  - [Portas que devemos nos preocupar](day-1/DescomplicandoKubernetes-Day1.md#portas-que-devemos-nos-preocupar)
  - [Tá, mas qual tipo de aplicação eu devo rodar sobre o k8s?](day-1/DescomplicandoKubernetes-Day1.md#tá-mas-qual-tipo-de-aplicação-eu-devo-rodar-sobre-o-k8s)
  - [Conceitos-chave do k8s](day-1/DescomplicandoKubernetes-Day1.md#conceitos-chave-do-k8s)
- [Aviso sobre os comandos](day-1/DescomplicandoKubernetes-Day1.md#aviso-sobre-os-comandos)
- [Minikube](day-1/DescomplicandoKubernetes-Day1.md#minikube)
  - [Requisitos básicos](day-1/DescomplicandoKubernetes-Day1.md#requisitos-básicos)
  - [Instalação do Minikube no GNU/Linux](day-1/DescomplicandoKubernetes-Day1.md#instalação-do-minikube-no-gnulinux)
  - [Instalação do Minikube no MacOS](day-1/DescomplicandoKubernetes-Day1.md#instalação-do-minikube-no-macos)
  - [kubectl: alias e autocomplete](day-1/DescomplicandoKubernetes-Day1.md#kubectl-alias-e-autocomplete)
  - [Instalação do Minikube no Microsoft Windows](day-1/DescomplicandoKubernetes-Day1.md#instalação-do-minikube-no-microsoft-windows)
  - [Iniciando, parando e excluindo o Minikube](day-1/DescomplicandoKubernetes-Day1.md#iniciando-parando-e-excluindo-o-minikube)
  - [Certo, e como eu sei que está tudo funcionando como deveria?](day-1/DescomplicandoKubernetes-Day1.md#certo-e-como-eu-sei-que-está-tudo-funcionando-como-deveria)
  - [Descobrindo o endereço do Minikube](day-1/DescomplicandoKubernetes-Day1.md#descobrindo-o-endereço-do-minikube)
  - [Acessando a máquina do Minikube via SSH](day-1/DescomplicandoKubernetes-Day1.md#acessando-a-máquina-do-minikube-via-ssh)
  - [Dashboard](day-1/DescomplicandoKubernetes-Day1.md#dashboard)
  - [Logs](day-1/DescomplicandoKubernetes-Day1.md#logs)
- [Microk8s](day-1/DescomplicandoKubernetes-Day1.md#microk8s)
  - [Requisitos básicos](day-1/DescomplicandoKubernetes-Day1.md#requisitos-básicos-1)
  - [Instalação do MicroK8s no GNU/Linux](day-1/DescomplicandoKubernetes-Day1.md#instalação-do-microk8s-no-gnulinux)
    - [Versões que suportam Snap](day-1/DescomplicandoKubernetes-Day1.md#versões-que-suportam-snap)
  - [Instalação no Windows](day-1/DescomplicandoKubernetes-Day1.md#instalação-no-windows)
    - [Instalando o Chocolatey](day-1/DescomplicandoKubernetes-Day1.md#instalando-o-chocolatey)
      - [Instalando o Multipass](day-1/DescomplicandoKubernetes-Day1.md#instalando-o-multipass)
    - [Utilizando Microk8s com Multipass](day-1/DescomplicandoKubernetes-Day1.md#utilizando-microk8s-com-multipass)
  - [Instalando o Microk8s no Mac](day-1/DescomplicandoKubernetes-Day1.md#instalando-o-microk8s-no-mac)
    - [Instalando o Brew](day-1/DescomplicandoKubernetes-Day1.md#instalando-o-brew)
    - [Instalando o Microk8s via Brew](day-1/DescomplicandoKubernetes-Day1.md#instalando-o-microk8s-via-brew)
- [Kind](day-1/DescomplicandoKubernetes-Day1.md#kind)
  - [Instalação no GNU/Linux](day-1/DescomplicandoKubernetes-Day1.md#instalação-no-gnulinux)
  - [Instalação no MacOS](day-1/DescomplicandoKubernetes-Day1.md#instalação-no-macos)
  - [Instalação no Windows](day-1/DescomplicandoKubernetes-Day1.md#instalação-no-windows-1)
    - [Instalação no Windows via Chocolatey](day-1/DescomplicandoKubernetes-Day1.md#instalação-no-windows-via-chocolatey)
  - [Criando um cluster com o Kind](day-1/DescomplicandoKubernetes-Day1.md#criando-um-cluster-com-o-kind)
    - [Criando um cluster com múltiplos nós locais com o Kind](day-1/DescomplicandoKubernetes-Day1.md#criando-um-cluster-com-múltiplos-nós-locais-com-o-kind)
- [k3s](day-1/DescomplicandoKubernetes-Day1.md#k3s)
- [Instalação em cluster com três nós](day-1/DescomplicandoKubernetes-Day1.md#instalação-em-cluster-com-três-nós)
  - [Requisitos básicos](day-1/DescomplicandoKubernetes-Day1.md#requisitos-básicos-2)
  - [Configuração de módulos de kernel](day-1/DescomplicandoKubernetes-Day1.md#configuração-de-módulos-de-kernel)
  - [Atualização da distribuição](day-1/DescomplicandoKubernetes-Day1.md#atualização-da-distribuição)
  - [Instalação do Docker e do Kubernetes](day-1/DescomplicandoKubernetes-Day1.md#instalação-do-docker-e-do-kubernetes)
  - [Inicialização do cluster](day-1/DescomplicandoKubernetes-Day1.md#inicialização-do-cluster)
  - [Configuração do arquivo de contextos do kubectl](day-1/DescomplicandoKubernetes-Day1.md#configuração-do-arquivo-de-contextos-do-kubectl)
  - [Inserindo os nós workers no cluster](day-1/DescomplicandoKubernetes-Day1.md#inserindo-os-nós-workers-no-cluster)
    - [Múltiplas Interfaces](day-1/DescomplicandoKubernetes-Day1.md#múltiplas-interfaces)
  - [Instalação do pod network](day-1/DescomplicandoKubernetes-Day1.md#instalação-do-pod-network)
  - [Verificando a instalação](day-1/DescomplicandoKubernetes-Day1.md#verificando-a-instalação)
- [Primeiros passos no k8s](day-1/DescomplicandoKubernetes-Day1.md#primeiros-passos-no-k8s)
  - [Exibindo informações detalhadas sobre os nós](day-1/DescomplicandoKubernetes-Day1.md#exibindo-informações-detalhadas-sobre-os-nós)
  - [Exibindo novamente token para entrar no cluster](day-1/DescomplicandoKubernetes-Day1.md#exibindo-novamente-token-para-entrar-no-cluster)
  - [Ativando o autocomplete](day-1/DescomplicandoKubernetes-Day1.md#ativando-o-autocomplete)
  - [Verificando os namespaces e pods](day-1/DescomplicandoKubernetes-Day1.md#verificando-os-namespaces-e-pods)
  - [Executando nosso primeiro pod no k8s](day-1/DescomplicandoKubernetes-Day1.md#executando-nosso-primeiro-pod-no-k8s)
  - [Verificar os últimos eventos do cluster](day-1/DescomplicandoKubernetes-Day1.md#verificar-os-últimos-eventos-do-cluster)
  - [Efetuar o dump de um objeto em formato YAML](day-1/DescomplicandoKubernetes-Day1.md#efetuar-o-dump-de-um-objeto-em-formato-yaml)
  - [Socorro, são muitas opções!](day-1/DescomplicandoKubernetes-Day1.md#socorro-são-muitas-opções)
  - [Expondo o pod](day-1/DescomplicandoKubernetes-Day1.md#expondo-o-pod)
  - [Limpando tudo e indo para casa](day-1/DescomplicandoKubernetes-Day1.md#limpando-tudo-e-indo-para-casa)

<!-- TOC -->



<!-- TOC -->

- [Descomplicando Kubernetes Day 2](day-2/DescomplicandoKubernetes-Day2.md#descomplicando-kubernetes-day-2)
  - [Sumário](day-2/DescomplicandoKubernetes-Day2.md#sumário)
- [Componentes do K8s](day-2/DescomplicandoKubernetes-Day2.md#componentes-do-k8s)
- [Principais Comandos](day-2/DescomplicandoKubernetes-Day2.md#principais-comandos)
- [Container Network Interface](day-2/DescomplicandoKubernetes-Day2.md#container-network-interface)
- [Services](day-2/DescomplicandoKubernetes-Day2.md#services)
  - [Criando um service ClusterIP](day-2/DescomplicandoKubernetes-Day2.md#criando-um-service-clusterip)
  - [Criando um service NodePort](day-2/DescomplicandoKubernetes-Day2.md#criando-um-service-nodeport)
  - [Criando um service LoadBalancer](day-2/DescomplicandoKubernetes-Day2.md#criando-um-service-loadbalancer)
  - [EndPoint](day-2/DescomplicandoKubernetes-Day2.md#endpoit)
- [Limitando Recursos](day-2/DescomplicandoKubernetes-Day2.md#limitando-recursos)
- [Namespaces](day-2/DescomplicandoKubernetes-Day2.md#namespaces)
- [Kubectl taint](day-2/DescomplicandoKubernetes-Day2.md#kubectl-taint)

<!-- TOC -->



<!-- TOC -->

- [Descomplicando Kubernetes Day 3](day-3/DescomplicandoKubernetes-Day3.md#descomplicando-kubernetes-day-3)
  - [Sumário](day-3/DescomplicandoKubernetes-Day3.md#sumário)
- [Deployments](day-3/DescomplicandoKubernetes-Day3.md#deployments)
  - [Filtrando por Labels](day-3/DescomplicandoKubernetes-Day3.md#filtrando-por-labels)
  - [Node Selector](day-3/DescomplicandoKubernetes-Day3.md#node-selector)
  - [Kubectl Edit](day-3/DescomplicandoKubernetes-Day3.md#kubectl-edit)
- [ReplicaSet](day-3/DescomplicandoKubernetes-Day3.md#replicaset)
- [DaemonSet](day-3/DescomplicandoKubernetes-Day3.md#daemonset)
- [Rollouts e Rollbacks](day-3/DescomplicandoKubernetes-Day3.md#rollouts-e-rollbacks)

<!-- TOC -->



<!-- TOC -->

- [Descomplicando Kubernetes Day 4](day-4/DescomplicandoKubernetes-Day4.md#descomplicando-kubernetes-day-4)
  - [Sumário](day-4/DescomplicandoKubernetes-Day4.md#sumário)
- [Volumes](day-4/DescomplicandoKubernetes-Day4.md#volumes)
  - [Empty-Dir](day-4/DescomplicandoKubernetes-Day4.md#empty-dir)
  - [Persistent Volume](day-4/DescomplicandoKubernetes-Day4.md#persistent-volume)
- [Cron Jobs](day-4/DescomplicandoKubernetes-Day4.md#cron-jobs)
- [Secrets](day-4/DescomplicandoKubernetes-Day4.md#secrets)
- [ConfigMaps](day-4/DescomplicandoKubernetes-Day4.md#configmaps)
- [InitContainers](day-4/DescomplicandoKubernetes-Day4.md#initcontainers)
- [Criando um usuário no Kubernetes](day-4/DescomplicandoKubernetes-Day4.md#criando-um-usuário-no-kubernetes)
- [RBAC](day-4/DescomplicandoKubernetes-Day4.md#rbac)
  - [Role e ClusterRole](day-4/DescomplicandoKubernetes-Day4.md#role-e-clusterrole)
  - [RoleBinding e ClusterRoleBinding](day-4/DescomplicandoKubernetes-Day4.md#rolebinding-e-clusterrolebinding)
- [Helm](day-4/DescomplicandoKubernetes-Day4.md#helm)
  - [Instalando o Helm 3](day-4/DescomplicandoKubernetes-Day4.md#instalando-o-helm-3)
  - [Comandos Básicos do Helm 3](day-4/DescomplicandoKubernetes-Day4.md#comandos-básicos-do-helm-3)

<!-- TOC -->



<!-- TOC -->

- [Descomplicando Kubernetes Day 5](day-5/DescomplicandoKubernetes-Day5.md#descomplicando-kubernetes-day-5)
  - [Sumário](day-5/DescomplicandoKubernetes-Day5.md#sumário)
- [Ingress](day-5/DescomplicandoKubernetes-Day5.md#ingress)

<!-- TOC -->



<!-- TOC -->
- [Descomplicando Kubernetes Day 6](day-6/DescomplicandoKubernetes-Day6.md#descomplicando-kubernetes-day-6)
  - [Sumário](day-6/DescomplicandoKubernetes-Day6.md#sumário)
- [Security Context](day-6/DescomplicandoKubernetes-Day6.md#security-context)
  - [Utilizando o security Context](day-6/DescomplicandoKubernetes-Day6.md#utilizando-o-security-context)
  - [Capabilities](day-6/DescomplicandoKubernetes-Day6.md#capabilities)
- [Manutenção do Cluster ETCD](day-6/DescomplicandoKubernetes-Day6.md#manutenção-do-cluster-etcd)
  - [O que preciso saber antes de começar?](day-6/DescomplicandoKubernetes-Day6.md#o-que-preciso-saber-antes-de-começar)
  - [O que é o ETCD?](day-6/DescomplicandoKubernetes-Day6.md#o-que-é-o-etcd)
  - [ETCD no Kubernetes](day-6/DescomplicandoKubernetes-Day6.md#etcd-no-kubernetes)
  - [Certificados ETCD](day-6/DescomplicandoKubernetes-Day6.md#certificados-etcd)
  - [Interagindo com o ETCD](day-6/DescomplicandoKubernetes-Day6.md#interagindo-com-o-etcd)
  - [Backup do ETCD no Kubernetes](day-6/DescomplicandoKubernetes-Day6.md#backup-do-etcd-no-kubernetes)
- [Dicas para os exames](day-6/DescomplicandoKubernetes-Day6.md#dicas-para-os-exames)

<!-- TOC -->
