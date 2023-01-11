# Sobrevivendo sem Docker em seu Computador, no Cluster Kubernetes e na sua Pipeline CI/CD

### `- Motivador para sobreviver sem o Docker`

Com o anúncio da Docker em 31 de Agosto de 2021, toda a comunidade Open Source do Kubernetes inclusive se movimentou para que o Docker não fosse o único Container Runtime Interface (CRI), e com isso introduziu nas últimas versões o crio e o containerd a partir da versão 1.24 do Kubernetes.

### `- Docker começará a cobrar por seu uso`

Ou seja, tudo dentro ou fora de orquestração de containers pode e vai ser cobrado, inclusive várias empresas podem ou já deve ter recebido a cobrança.

Agora vai de sua decisão de financeira, técnica e de arquitetura se você segue com isso ou não.

Link do Anúncio de cobrança da Docker em seu blog:  
[https://www.docker.com/blog/updating-product-subscriptions/](https://www.docker.com/blog/updating-product-subscriptions/)

Mas o mundo é feito por boa alternativas, como o **Podman** que vamos seguir por aqui com ele.

### `- Apresentando Podman`

Uma ferramenta de gerenciamento de container compatível com [OCI (Open Container Interface),](https://github.com/opencontainers) o qual que oferece recursos semelhantes, como o Docker, para gerenciamento de contêineres.

> Um dos melhores recursos do podman é a capacidade de executar containers em **modo rootless**

Um container em rootless é um conceito de execução e gerenciamento de containers sem privilégios de root.

Do ponto de vista da segurança, os containers adicionam uma camada adicional de segurança ao não permitir o acesso root, mesmo que o containers seja comprometido por um invasor.

Leia mais sobre esse assunto, no link abaixo:  
[https://blog.aquasec.com/rootless-containers-boosting-container-security](https://blog.aquasec.com/rootless-containers-boosting-container-security)

---

Alguns pontos bem interessantes:

* O Podman também é sem daemon (ao contrário do docker), o que significa que não possui um daemon e interage diretamente com o [runC](https://github.com/opencontainers/runc), o qual lida com todo o gerenciamento e criação dos containers de acordo com a especificação do OCI.
    
* Recurso interessante e avançado do podman é a execução de containers em Pods. Semelhante aos pods do Kubernetes, você pode criar pods de vários containers localmente usando o Podman.
    
* Você pode exportar o pod do podman como um manifesto do Kubernetes e usar um manifesto do pod do Kubernetes para implantar um pod do podman.
    

### `- Instalando e usando o Podman`

Acesse a documentação oficial de instalação do Podman. Aqui você encontrará todos os comandos de instalação para Windows, MAC e Linux, no seguinte link abaixo:

[https://podman.io/getting-started/installation](https://podman.io/getting-started/installation)

Após instalado, segue alguns comandos de fácil uso com o Podman.

### `- Rodando um Container a partir da imagem`

O comando run cria um container de uma determinada imagem e o executa. Vamos executar a imagem do CentOS que criamos anteriormente

```bash
podman run --name centos-pod -p 80:80 -dit centos
```

Este comando primeiro verifica se há uma imagem local disponível para o CentOS. Se a imagem não estiver presente localmente, ele tenta extrair a imagem dos registros que foram configurados. Se a imagem não estiver presente nos registros, ele mostra um erro sobre incapaz de encontrar a imagem.

O comando de execução acima especifica o mapeamento da porta 80 exposta do container para a porta 80 do host e o parâmetro dit especifica a execução do container no modo detached e interativo. O id do container criado será a saída.

### `- Testando na prática`

Clone do GIT, o qual seria um simples Projeto de Votação para realizarmos os testes da seguinte forma:

```bash
git clone https://github.com/andrebassi/gitops-multienv-vote.git
```

Logo após entre na seguinte pasta desse projeto

```bash
cd gitops-multienv-vote/vote/
```

Agora vamos gerar a imagem do mesmo com o Podman:

```bash
podman build -t ttl.sh/multivote:1h \
  --build-arg NODE_ENV=18 \
  --build-arg VERSION=1.0.0 \
  --build-arg WEBSITE_PORT=80 .
```

Verificamos a imagem criada

```bash
podman images | grep multivote
```

Você obterá a saída:

```bash
ttl.sh/multivote  1h  0256fc501009  10 hours ago  262 MB
```

Agora efetauremos o push ao [serviço de Registry efêmero e temporário, denominado ttl.sh](https://ttl.sh/):

```bash
podman push ttl.sh/multivote:1h
```

Agora podemos testar o container criado no daemon local do Podman da seguinte forma:

```bash
podman run --name multi-vote -p 8080:80 -dit ttl.sh/multivote:1h
```

E em seguida visite [http://localhost:8080](http://0.0.0.0:8080)

Você visualizará o sistema de Votos rodando em seu browser preferido:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673366570871/4bd30863-6788-47bb-ae67-cf23b5e3afb5.png align="center")

### `- No Cluster Kubernetes`

Um comando bem interessante do Podman seria simplemente pegar um dos containers e exportá-lo para uma configuração YAML compatível com Kubernetes, da seguinte forma:

Com o container em execução o Podman pode gerar o manifesto da seguinte forma

```bash
podman generate kube multi-vote -f pod-multi-vote.yaml
```

O arquivo **pod-multi-vote.yaml** foi criado com um manifesto no formato YAML com a criação de seu `Pod` para o Kubernetes conforme demonstra abaixo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    io.kubernetes.cri-o.TTY/multi-vote: "true"
    io.podman.annotations.autoremove/multi-vote: "FALSE"
    io.podman.annotations.init/multi-vote: "FALSE"
    io.podman.annotations.privileged/multi-vote: "FALSE"
    io.podman.annotations.publish-all/multi-vote: "FALSE"
    org.opencontainers.image.base.digest/multi-vote: sha256:45f53e19ff6fad6de58f62dc79a0a4b1499685717333b1e41b9b6bef
    org.opencontainers.image.base.name/multi-vote: docker.io/library/node:18.2-slim
  creationTimestamp: "2023-01-10T07:16:28Z"
  labels:
    app: multi-vote-pod
  name: multi-vote-pod
spec:
  automountServiceAccountToken: false
  containers:
  - args:
    - node
    - index.js
    image: ttl.sh/multivote:1h
    name: multi-vote
    ports:
    - containerPort: 80
      hostPort: 8080
    securityContext:
      capabilities:
        drop:
        - CAP_MKNOD
        - CAP_NET_RAW
        - CAP_AUDIT_WRITE
    stdin: true
    tty: true
  enableServiceLinks: false
```

Se aplicarmos esse Manifesto no Kubernetes teremos 1 Pod criado:

```bash
NAME             READY   STATUS    RESTARTS   AGE
multi-vote-pod   1/1     Running   0          24s
```

Os logs do Pod:

```bash
env.NODE_ENV: 18
env.VERSION: 1.0.0
env.WEBSITE_PORT: 80
listening on port 80
```

Parte do Describe do Pod:

```bash
Containers:
  multi-vote:
    Container ID:  containerd://6c750387ce3dd232e71001d8e6c7924bf35041aa0eb40b63ee295c27
    Image:         ttl.sh/multivote:1h
    Image ID:      ttl.sh/multivote@sha256:5f0cbc5504cc2934fe87a9e865a5df8b48ac96bbe6f29
    Port:          80/TCP
    Host Port:     8080/TCP
    Args:
      node
      index.js
    State:          Running
      Started:      Tue, 10 Jan 2023 13:30:24 -0300
    Ready:          True
    Restart Count:  0
```

Para esse laboratório de exemplos usamos o Container Runtime Interface (CRI) como **containerd://1.6.14**

Mais detalhes sobre o containerd, vide no link abaixo:

[https://containerd.io/](https://containerd.io/)

E também seria totalmente compatível com o crio-o:  
[https://cri-o.io/](https://cri-o.io/)

### `- Na pipeline CI/CD do Gitlab`

Podman é uma versão reimplementada do Docker da RedHat. Ele suporta as mesmas opções de linha de comando, mas tem uma arquitetura fundamentalmente diferente: ao contrário do Docker, não há daemon por padrão. A CLI faz todo o trabalho sozinha.

Isso significa que podemos fazer uma configuração GitLab CI muito mais simples, sem o serviço executando o daemon:

```yaml
stages:
  - build

# Build and push the Docker image to the GitLab image registry
# using Podman.
podman-build:
  stage: build

  image:
    name: quay.io/podman/stable

  script:
    # GitLab has a built-in Docker image registry, whose
    # parameters are set automatically. You can use some
    # other Docker registry though by changing the login and
    # image name.
    - podman login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - podman build -t "$CI_REGISTRY_IMAGE:podman" .
    - podman push "$CI_REGISTRY_IMAGE:podman"
```

Veja que continua sendo tão simples que tomar o uso do DindD (Docker in Docker) como serviço no Gitlab:

### `- Na pipeline do Github`

Artigos demonstram como usar:

[https://www.anmalkov.com/blog/build-a-ci-workflow-in-github-actions-with-buildah-and-podman](https://www.anmalkov.com/blog/build-a-ci-workflow-in-github-actions-with-buildah-and-podman)

[https://dev.to/anmalkov/how-to-live-without-docker-for-developers-part-4-ci-workflow-in-github-actions-with-buildah-and-podman-kk4](https://dev.to/anmalkov/how-to-live-without-docker-for-developers-part-4-ci-workflow-in-github-actions-with-buildah-and-podman-kk4)

### `- Trivy com o Podman`

O **Trivy**, uma ferramenta simples de código aberto que é mantida pelo [Aqua Security](https://www.aquasec.com/) Essa ferramenta é usada para varredura de vulnerabilidade abrangente para containers e outros artefatos.

Para realizar sua instalação, um processo bem simples, siga os passos abaixo:

```bash
export TRIVY_VERSION=${TRIVY_VERSION:-v0.36.1} \
&& curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin ${TRIVY_VERSION}
```

Após o cli do Trivy instalado, execute na imagem criada anteriormente pelo Podman da seguinte forma:

```bash
trivy --debug image --severity MEDIUM,HIGH,CRITICAL ttl.sh/multivote:1h
```

Um report no bash será apresentando igual a esse como exemplo, indicando qual lib que está rodando no container que precisa ser atualizado, o qual tanto no S.O e Aplicativo rodando queconsta como vulneribilidade CVE cadastrado, sua severidade e qual versão você deverá atualizar para isso ser corrigido.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673370882047/ec9efdad-4894-40e3-b02d-09bc896fd4bf.png align="center")

### `- Considerações importantes sobre a segurança do container com Podman`

Lembrando que um container é um ambiente isolado utilizado para empacotar aplicações, onde containers têm o objetivo de segregar e facilitar a portabilidade de aplicações em diferentes ambientes.

Um container contém um conjunto de processos que são executados a partir de uma imagem, imagem esta que fornece todos os arquivos necessários.

Os containers compartilham o mesmo kernel e isolam os processos da aplicação do restante do sistema operacional.

A idéia é que cada container assuma apenas uma responsabilidade. Nos containers, você divide a responsabilidade isolando os processos de cada aplicação, garantindo assim que nenhum processo possa influenciar no funcionamento dos demais processos.

Por exemplo você pode ter dentro de um container um Alpine, Ubuntu, CentOS e entre outros, assim rodando de forma segregada.

É isso que devemos por a segurança a prova juntamente com a aplicação que rodará em conjunto em seu ciclo.

É isso que o Trivy faz muito bem, te apontar além da aplicação, todo o ecossistema do container produzido pelo Podman.

Para conhecer mais sobre o Trivy, navegue em sua documentação no Link:

[https://aquasecurity.github.io/trivy/v0.36/](https://aquasecurity.github.io/trivy/v0.36/)