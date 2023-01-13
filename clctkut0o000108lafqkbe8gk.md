# Auto Instrumentação do OpenTelemetry

Neste artigo, gostaria de apresentar o novo recurso do OpenTelemetry Operator que simplifica significativamente as cargas de trabalho de instrumentação implantadas no Kubernetes.

A instrumentação é o processo mais tedioso ao implantar uma solução de observabilidade. Existem várias abordagens de como instrumentar e aplicar:

* **Manual**: o código-fonte é explicitamente instrumentado, por exemplo, usando a API OpenTelemetry ou usando bibliotecas de instrumentação pré-construídas que são vinculadas no tempo de compilação;
    
* **Automático**: o aplicativo é instrumentado sem nenhuma modificação de código e a recompilação do aplicativo não é necessária;
    

A instrumentação automática esteve por muito tempo disponível apenas como uma tecnologia proprietária oferecida por vários fornecedores de APM/Observabilidade.

A OpenTelemetry mudou esse paradigma e disponibilizou essa tecnologia em código aberto. Os usuários obtêm instrumentação neutra de fornecedor para evitar o bloqueio do fornecedor para a parte mais crucial da integração de observabilidade.

No entanto, implantar auto instrumentação em escala ou validar prova de valor ainda pode ser um problema bem árduo.

Os aplicativos modernos são empacotados em imagens imutáveis, portanto, adicionar instrumentação a imagens já construídas requer a reconstrução do container.

Reconstruir um grande número de imagens de aplicativos requer um grande investimento. Vamos dar uma olhada em como esse problema pode ser resolvido no Kubernetes.

### `Instrumentação no Operator do OpenTelemetry`

O OpenTelemetry Operator introduziu o recurso customizado de Instrumentação que define a configuração para OpenTelemetry SDK e instrumentação. A instrumentação é habilitada quando um CR de instrumentação está presente no cluster e um namespace ou o workload recebe uma annotation:

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: my-instrumentation
spec:
  exporter:
    endpoint: http://otel-collector:4317
  propagators:
    - tracecontext
    - baggage
    - b3
  sampler:
    type: parentbased_traceidratio
    argument: "0.25"
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest
  nodejs:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:latest
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:latest
EOF
```

No momento, a instrumentação é compatível com as linguagens Java, NodeJS e Python. A instrumentação é habilitada quando a seguinte annotation é aplicada a um workload ou uma namespace.

* [`instrumentation.opentelemetry.io/inject-java`](http://instrumentation.opentelemetry.io/inject-java)`: "true"` — Java
    
* [`instrumentation.opentelemetry.io/inject-nodejs`](http://instrumentation.opentelemetry.io/inject-nodejs)`: "true"` — NodeJS
    
* [`instrumentation.opentelemetry.io/inject-python`](http://instrumentation.opentelemetry.io/inject-python)`: "true"` — Python
    

Depois que a annotation é aplicada, o operador injeta as bibliotecas de autoinstrumentação OpenTelemetry no container do aplicativo e configura a instrumentação para exportar os dados para um endpoint definido no Instrumentation CR.

### `Exemplo de Java com Spring Boot`

O projeto com desenvolvido em **Java Spring Boot,** denomiado **PetClinic** é um bom exemplo para esse caso e o mesmo pode ser encontra no github através do seguinte link:

[https://github.com/spring-projects/spring-petclinic](https://github.com/spring-projects/spring-petclinic)

Agora vamos realizar o deploy do aplicativo `Java Spring Petclinic` que será instrumentado e reportará os dados para um coletor OpenTelemetry. O coletor registrará spans na saída padrão.

Crie a seguinte Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-petclinic
spec:
  selector:
    matchLabels:
      app: spring-petclinic
  replicas: 1
  template:
    metadata:
      labels:
        app: spring-petclinic
      annotations:
        sidecar.opentelemetry.io/inject: "true"
        instrumentation.opentelemetry.io/inject-java: "true"
    spec:
      containers:
      - name: app
        image: ghcr.io/pavolloffay/spring-petclinic:latest
```

e aplique a annotation de instrumentação:

```bash
kubectl patch deployment.apps/spring-petclinic -p '{"spec": {"template": {"metadata": {"annotations": {"instrumentation.opentelemetry.io/inject-java": "true"}}}}}'
```

Após a aplicação da annotation, o pod spring-petclinic será reiniciado e o pod recém-iniciado será instrumentado com a autoinstrumentação Java OpenTelemetry. Os dados serão informados em formato OTLP ao coletor no endereço [http://otel-collector:4317](http://otel-collector:4317).

O trecho de código a seguir implanta um coletor OpenTelemetry que registra rastreamentos na saída padrão.

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel
spec:
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
    processors:

    exporters:
      logging:

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: []
          exporters: [logging]
EOF
```

Agora podemos encaminhar a porta HTTP do aplicativo:

```bash
kubectl port-forward deployment.apps/spring-petclinic 8080:8080
```

E explorar o aplicativo no navegador WEB.

Os spans devem ser relatados ao coletor OpenTelemetry. Os períodos relatados podem ser visualizados por meio de

```bash
kubectl logs deployment.apps/otel-collector
```

A instrumentação automática é um software bastante complicado, a implementação depende do idioma.

Em Java, a auto-instrumentação é chamada de Javaagent e faz manipulação de bytecode que injeta pontos de instrumentação em caminhos de código específicos.

Em seguida, os pontos de instrumentação criam dados de telemetria (por exemplo, traços) quando o código é executado. A auto-instrumentação é sempre suportada por um conjunto definido de frameworks ou APIs. Esta é a lista de estruturas e servidores de aplicativos suportados para Java.

### `Como a lógica de injeção é implementada?`

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673494890190/32add0ad-3d59-4351-8a65-0911ccd78d33.png align="center")

Você pode se perguntar como a injeção de auto instrumentação é implementada e principalmente como o operador pode configurar o aplicativo para usá-lo. Deixe-me explicar para Java, no entanto, conceitos semelhantes também são usados para outros tempos de execução.

O operador OpenTelemetry implementa o `Webhook admission mutating` que é invocado quando o objeto Pod é criado ou atualizado. O webhook modifica o objeto Pod para injetar bibliotecas de autoinstrumentação no container.

Pois ele configura **OpenTelemetry SDK** e tempo de execução, neste caso, Java virtual machine (**JVM**) para usar a auto instrumentação.

A biblioteca de instrumentação automática (`Javaagent`) é injetada no container por meio de um `container init` que copia o `Javaagent` em um volume montado no container do aplicativo. A configuração do SDK é feita injetando variáveis de ambiente no container.

Agora, a etapa final é configurar a `JVM` para usar o `Javaagent`. Isso é feito configurando a variável de ambiente `JAVA_TOOL_OPTIONS` para usar o `Javaagent`.

Mais detalhes pode ser encontrada em:  
[https://github.com/open-telemetry/opentelemetry-operator#opentelemetry-auto-instrumentation-injection](https://github.com/open-telemetry/opentelemetry-operator#opentelemetry-auto-instrumentation-injection)

### `OpenTelemetry no Google`

Vídeo que demonstra como podemos fazer auto instrumentação utilizando os serviços do Google Cloud.

%[https://www.youtube.com/watch?v=RuyUXBOdjGI] 

### `Mais um exemplo fo Google....`

Outro tutorial bem bacana para se aprofundar no assunto, veja no link abaixo:

[https://cloud.google.com/blog/topics/developers-practitioners/easy-telemetry-instrumentation-gke-opentelemetry-operator](https://cloud.google.com/blog/topics/developers-practitioners/easy-telemetry-instrumentation-gke-opentelemetry-operator)

### `Serviço Saas de OpenTelemetry`

A Aspecto oferece um serviço SaaS bem interessante de rastreamento distribuído de ponta a ponta com amostragem integrada inteligente com intuiro de reduzir o custo de telemetria.

Mais sobre o serviço veja o link do serviço:

[https://www.aspecto.io/](https://www.aspecto.io/)

### `Aprendendo mais sobre...`

No serviço acima sugerido, a Aspecto forneceum treinamento gratúito muito bom sobre o assunto, veja mais em:

[https://www.aspecto.io/opentelemetry-bootcamp/](https://www.aspecto.io/opentelemetry-bootcamp/)