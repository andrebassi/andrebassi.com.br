# Knative | Scale to Zero

O Knative tem como objetivo disponibilizar implementações reutilizáveis de padrões e melhores práticas de codificação, com os seguintes componentes:

* [Build](https://github.com/knative/build) - Orquestração de builds para containers;
    
* [Eventing](https://github.com/knative/eventing) - Gerenciamento e entrega de eventos;
    
* [Serving](https://github.com/knative/serving) - Computação request-driven que pode escalar para zero;
    

### `Build`

O componente de build do Knative é uma extensão do [Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) e utiliza primitivas existentes para permitir a execução de builds de container no cluster, a partir do código fonte (baseado no [Kaniko](https://www.infoq.com/news/2018/04/kaniko-container-image-builder), lançado anteriormente). O objetivo é utilizar os recursos nativos do Kubernetes para obter código fonte de um repositório, fazer o build dentro de uma imagem de container, e então rodar esta imagem. Porém, a documentação diz que o usuário final do framework [ainda é responsável](https://github.com/knative/build) por desenvolver os componentes correspondentes que executam a maioria destas funções.

> Até o momento, um build de Knative não possui uma solução CI/CD completa, mas por outro lado possui um building block de baixo nível que foi desenhado para permitir integração e utilização em sistemas maiores.

### `Eventing`

O sistema de eventos foi desenhado para [resolver uma série de necessidades comuns](https://github.com/knative/docs/tree/master/eventing) do desenvolvimento de software cloud native: os serviços são fracamente pareados durante o desenvolvimento e seus deploys são feitos de forma independente; um produtor pode gerar eventos antes que haja um consumidor ouvindo, e um consumidor pode expressar interesse em um evento ou classe de eventos que ainda não está sendo produzida; e serviços podem ser conectados para criar novas aplicações sem modificar o produtor ou o consumidor, com a habilidade de selecionar um subconjunto específico de eventos de um produtor específico.

A documentação mostra que estes objetivos de design são consistentes com os [objetivos de design do CloudEvents](https://github.com/cloudevents/spec/blob/master/spec.md#design-goals), uma especificação comum para interoperabilidade cross-service que está sendo desenvolvida pelo CNCF Serverless WG. A documentação dentro do repositório de Eventing deixa claro que ainda é um trabalho em andamento, e que existe uma [lista de problemas conhecidos](https://github.com/knative/eventing/issues?q=is:issue+is:open+label:%22Known+Issue%22).

### `Serving`

A documentação do [Knative Serving](https://github.com/knative/serving) mostra que esta parte do framework foi feita sobre o Kubernetes e Istio para suportar deploy e disponibilização de aplicações e funções serverless. O objetivo é prover primitivas middleware que permitem: o deploy rápido de containers serverless; scaling automático "up and down to zero"; roteamento e programação de redes para componentes Istio; e snapshots point-in-time de código e configurações que foram utilizadas durante o deploy.

[Joe Beda](https://www.linkedin.com/in/jbeda/), Pai do Kubernetes, Fundador e CTO na Heptio, escreveu no twitter os [potenciais beneficios](https://twitter.com/jbeda/status/1021798510025306112) do componente de serving:

> Uma das partes mais interessantes do KNative é o "scale to zero". Isto é feito roteando as requisições para um "atuador" que guarda requests, escala o backend e então prossegue.
> 
> Eu estava esperando que alguém construísse isto.

### `Autoscaling`

O Knative fornece dimensionamento automático para os pods K8s gerenciados pelo Knative Services (CRD).

O Knative implementa uma solução de dimensionamento automático chamada Knative Pod Autoscaler (KPA) que você pode usar com seus aplicativos, fornecendo os recursos abaixo:

* **Scale-To-Zero**:  
    O Knative usa o Knative Pod Autoscaler (KPA) por padrão. Com o KPA, você pode dimensionar seu aplicativo para zero pods, se o aplicativo não estiver recebendo nenhum tráfego.
    
* **Concurrency**:  
    Você pode usar a configuração de concorrência para determinar quantas conexões simultâneas seus pods podem processar a qualquer momento. Se o número de solicitações exceder o limite para cada pod, o Knative aumentará o número de pods.
    
* **Requests Per Second (RPS)**:  
    Também é possível usar o Knative para definir quantas solicitações por segundo cada pod pode manipular. Se o número de solicitações por segundo exceder o limite, o Knative aumentará o número de pods.
    

Você também pode usar o Horizontal Pod Autoscaler (HPA) com Knative, mas HPA e KPA não podem ser usados juntos para o mesmo serviço. (O HPA não é instalado pelo Knative, caso queira utilizá-lo, é necessário instalá-lo separadamente).

O KPA é usado por padrão, mas você pode controlar qual tipo de escalador automático usar, por meio de anotações nas definições de serviço.

HPA:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: sample-svc
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/class: "hpa.autoscaling.knative.dev"
```

KPA:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: sample-svc
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/class: "kpa.autoscaling.knative.dev"
```

Usando uma abordagem semelhante, você pode definir o tipo de métrica que deseja usar para dimensionar automaticamente seu serviço e também determinar a meta a ser alcançada para acioná-la.

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: sample-svc
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/class: "kpa.autoscaling.knative.dev"
        autoscaling.knative.dev/metric: "concurrency"
        autoscaling.knative.dev/target: "50"
```

Para maiores detalhes, consulte a documentação no link:

[https://knative.dev/docs/serving/autoscaling/](https://knative.dev/docs/serving/autoscaling/)

### `Divisão de Tráfego e Revisões`

A Revisão Knative é semelhante a uma tag ou label e é imutável. Cada Revisão Knative tem uma Implementação Kubernetes correspondente associada a ela; ele permite que o aplicativo seja revertido para qualquer uma das revisões anteriores.

Você pode ver a lista de revisões disponíveis executando o comando `kn revisions list` por exemplo.

Com as revisões, talvez você queira implantar aplicativos usando padrões de implantação comuns, como Canary ou blue-green. Você precisa ter mais de uma revisão de um serviço para usar esses padrões.

O serviço hello como ilustra como exemplo abaixo já possui duas revisões denominadas **hello-world** e **hello-coder** respectivamente.

Você pode dividir o tráfego em 50% para cada revisão usando o seguinte comando:

```bash
kn service update hello --traffic hello-world=50 \
--traffic hello-coder=50
```

### `Vídeo da DevOps Toolkit`

%[https://www.youtube.com/watch?v=VjI5WDOhAwk]