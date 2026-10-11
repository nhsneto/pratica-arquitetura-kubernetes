# Aponti Academy - Prática Arquitetura Kubernetes

# O que é Kubernetes

É o Orquestrador de Containers mais famoso do mundo!
Ele é Open Source, ou seja, gratuito se você utilizar na sua máquina.

## O que é Orquestração de Containers?

É o ato de você dar super poderes aos seus containers.
Ele **garante a disponibilidade da aplicação**, o **Escalonamento horizontal** (HPA) e etc.

> [!NOTE]
> Você tem uma aplicação que faz leitura de disco e pode ter um IOPS muito alto, o que pode derrubar a aplicação. Nesse caso, você teria que reiniciá-la manualmente, pois não está em um orquestrador de containers.

> [!TIP]
> Aplicações Java têm um startup lento por natureza. Quando seu Load Balancer inicia uma instância, a aplicação demora para iniciar completamente, mas as requisições já começam a chegar assim que o container está em pé. A aplicação não consegue responder porque ainda está iniciando, gerando vários erros.
>
> Com o Kubernetes, você pode definir um tempo de espera para que o container só comece a receber tráfego quando a aplicação estiver totalmente iniciada.

## Recursos que temos no kubernetes:

 **Pods**: É a menor unidade do Kubernetes.
Ele representa **um ou mais containers rodando juntos** no mesmo ambiente.
Kubernetes **não gerencia containers diretamente**, ele gerencia **Pods**.
 
 
 Dentro dos Pods contém os **containers que são gerenciados pelo Container Runtime.**
 
 **Kubelet**: é um componente que roda em cada nó do cluster. Ele solicita a criação do container através de um request para o Container Runtime
 
 **Container Runtime Interface**: é a parte da comunicação entre o Container Runtime e o Kubernetes. O Kubelet não consegue se comunicar diretamente sem um padrão (o CRI é esse padrão)
 
 **Container Runtime**: recebe o pedido do Kubelet e interage com o Kernel do Linux para criar os containers

```yaml
apiVersion: v1
kind: Pod                        #tipo do manifesto. no caso, pod
metadata:
  name: meu-pod                 # nome do pod
spec:                          #especificações do container
  containers:                    # container dentro do manifesto, as especificações
    - name: app                   # nome do container
      image: nginx                # aqui é a imagem do container, no caso, nginx (se vc tiver imagem no docker hub, ECR ou outros, é só colocar aqui)
      ports:
        - containerPort: 80     # aqui é a porta onde esse container vai escutar
```
Isso se chama Manifesto: é um arquivo YAML que vai representar o deploy do container no Kubernetes

> [!CAUTION]
> Atenção: Criar um manifesto tipo pod assim como está acima é chamado de Standalone e NÃO é a melhor prática, pois não terá disponibilidade se alguém vier a deletar esse pod.

## Deployment

Com o Deployment você diz ao Kubernetes quantas replicas de Pod você quer e ele criará. Ele garante sempre que o número de Pods esteja correto. *(mesmo se alguém vier manualmente e deletar o Pod)*

```yaml
apiVersion: apps/v1
kind: Deployment                #tipo do manifesto, no caso, deployment
metadata:
  name: nginx-deployment        #nome do deployment
  labels:
    app: nginx
spec:
  replicas: 3                  # aqui eu vou ter 3 replicas, 3 pods
  selector:                    #significa que esse deployment vai gerenciar todos os pods que tiverem a label "app: nginx"
    matchLabels:
      app: nginx
  template:                    #aqui, todos as replicas que esse deployment criar, eles irão ter a label "app: nginx"
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

> [!TIP]
> Essa é a melhor prática de se criar um manifesto. Usando o Deployment pois terá alta disponibilidade. Caso deletem o pod, o deployment subirá automaticamente.

## Deployment e  HPA (Horizontal Pod Autoscaler) juntos

O Deployment continua sendo o dono dos Pods e o HPA é que dá uma ordem para o Deployment

- **Sem HPA:** Você escreve no manifesto: `replicas: 3`. O Deployment garante que esses 3 estejam sempre vivos. Se você quiser 5, tem que ir lá e mudar o arquivo na mão.
- **Com HPA:** Você diz ao HPA: *"Olha, o limite mínimo é 2 e o máximo é 10"*. O HPA passa a **controlar** o campo `replicas` do Deployment de forma automática.

Não pense que o Deployment "perde a utilidade" depois que os Pods sobem. Ele tem funções vitais que o HPA não faz:

- **Self-Healing (Auto-cura):** Se um Pod travar ou o processo interno morrer, o Deployment percebe e cria um novo. O HPA não liga para Pods mortos, ele só liga para a média de uso de CPU/RAM.
- **Rollouts (Atualizações):** Quando você muda a versão da sua imagem (do `v1` para o `v2`), é o Deployment que mata os Pods velhos um por um e sobe os novos sem tirar o site do ar.
- **Rollback:** Se a versão nova der erro, o Deployment é quem sabe como voltar para a versão anterior rapidamente.

## Replica Set

Existe uma camada entre o Deployment e os Pods, essa camada entre eles se chama Replica Set. 
Quando você cria o deployment, automaticamente cria o Replica Set também. *(Na vida real, você raramente criará um ReplicaSet diretamente. Em vez disso, você usará um Deployment.)*
Ele vai **gerenciar a quantidade de pods em uma determinada versão.**
Ou seja, quando você trocar a versão da imagem do container em um Deployment, um novo Replica Set é criado com a nova versão e vai desligando o antigo aos poucos.

> [!NOTE]
> Isso permite utilizarmos Fast Rollback (voltar as minhas aplicações para a versão anterior) sem precisar subir toda a aplicação novamente caso uma atualização não saia como o desejado.

> [!TIP]
> O pulo do gato é que o Kubernetes não deleta o ReplicaSet antigo da `v1`. Ele o mantém guardado, mas com o número de réplicas configurado para 0.
>
> Se a `v2` apresentar bugs, você executa um comando: `kubectl rollout undo deploy/(nome do deployment) --revision (versão que deseja)`. O Kubernetes simplesmente diz ao ReplicaSet da `v1`: "Ei, volte a ter 3 réplicas" e ao da `v2`: "Reduza para 0"

## Service

Ele **balanceia o tráfego para os Pods existentes**. Se eu tiver 200 Pods, o Service vai fazer com que o tráfego vá para esses 200 e não para um único. *(Ele funciona como um "load balancer interno", distribuindo requisições entre os Pods.)*
O Load Balancer se comunica com o Service e ele se comunica com os Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: minha-app-service
spec:
  selector:
    app: minha-app
  ports:
    - protocol: TCP
      port: 80                 # porta do Service
      targetPort: 3000         # porta do container
```
## Tipos de Service

## Tipos de Service

<details>
<summary><b>ClusterIP</b></summary>

<br>

Ele cria um IP interno fixo dentro do cluster onde só pode ser acessado pelos microsserviços dentro do cluster.

É com o **ClusterIP** que ocorre a comunicação entre o Frontend e o Backend, Backend e o Banco de Dados.

**É o mais usado em produção.**

*(Caso não seja especificado nenhum tipo de service, o padrão será ClusterIP)*

</details>

<details>
<summary><b>NodePort</b></summary>

<br>

Com o NodePort a gente <u>especifica uma porta para expor aquele serviço</u> do nosso nó. Ele abre uma porta específica em todos os Nós (Nodes) do cluster.
Portas: Por padrão, o K8s reserva o intervalo **30000-32767**.

</details>

<details>
<summary><b>LoadBalancer</b></summary>

<br>

Ele se integra com algum provedor Cloud e cria um Load Balancer para redirecionar o tráfego externo.
>[!TIP]
> Em produção, é comum usar **ClusterIP** para os serviços e um **Ingress Controller** para gerenciar o tráfego externo de forma mais inteligente e econômica do que criar um LoadBalancer para cada Service.

</details>

<details>
<summary><b>ExternalName</b></summary>

<br>

Mapeia o serviço para o conteúdo do campo `externalName` (ex: um nome de domínio CNAME).

</details>