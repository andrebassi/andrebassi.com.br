# Usando Argo Workflow para orquestrar o NMAP a escanear milhares de IPs do range da AWS

Argo é perfeito para tarefas repetidas e worklows que podem ser reutilizados apenas fornecendo um parâmetro diferente.

Hoje vamos demonstrar como fase de como explorar mais sobre no ambíto da seghurança da informação.

`💡 Para este exemplo, presumimos que você entenda de Argo e esteja confortável com os modelos envolvidos.`

Vamos obter uma lista dos IPs atuais da AWS e executar o NMAP consequentemente em todos.

Definimos o seguinte manifesto com o `Kind Workflow`:

```yaml
apiVersion: argoproj.io/v1alpha1 
kind: Workflow
metadata:
  generateName: scan-nmap-aws-
  namespace: "argo"
spec:
  entrypoint: entry
  volumeClaimTemplates:
    - metadata:
        name: workdir
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi 
 parallelism: 10
```

O que fizemos acima?

Criamos um `workflow` vazio que gerará nomes semelhantes a `scan-nmap-aws-XXXXX`

Colocamos no namespace `argo` e criamos um `VolumeClaim` chamado `workdir` para nossos próximos passos.

### `Agora precisamos definir alguns passos...`

Se vamos escanear toda a AWS, precisamos obter esses intervalos de IP primeiramente.

Como podemos fazer isso? Bem, acredite ou não, a AWS fornece isso para nós!

```yaml
- name: get-ips
  script:
    image: byteknight/alpine-curl-jq:latest
    command: [sh, -c]
     args:
      [
        'curl -s ''https://ip-ranges.amazonaws.com/ip-ranges.json'' | jq ''[.prefixes[] | select(.service | contains("EC2")) | .ip_prefix]'' > /tmp/ips.json',
      ]
   outputs:
     parameters:
     - name: ips
       valueFrom:
         path: /tmp/ips.json
```

Agora temos os IPs de que precisamos, mas precisamos escaneá-los via NMAP em outra etapa.

### `Como vamos fazer isso?`

Enviando IPs ao NMAP

```yaml
- name: run-nmap
  inputs:
    parameters:
      - name: target
    container:
      image: securecodebox/nmap:latest
      securityContext:
	    privileged: true
	    allowPrivilegeEscalation: true
      command: [nmap]
      args:
        [
            "-sV",
            "-vv",
            "-T5",
            "-A",
            "-n",
            "--min-hostgroup",
            "100",
            "--min-parallelism",
            "200",
            "-Pn",
            "-oX",
            "/mnt/data/{{}}.xml",
            "{{inputs.parameters.target}}",
        ]
      volumeMounts:
        - name: workdir
          mountPath: /mnt/data
```

O container NMAP será criado com as opções especificadas e consequentemente executará sob os IPS destinos fornecidos na lista baixada.

Agora precisamos combinar todas as etapas e criar uma etapa de entrada principal, onde lidará com a passagem dos parâmetros e a ordenação das etapas.

### `Por uma questão de exemplo, este é o script inteiro até este ponto:`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: scan-nmap-aws-
  namespace: "argo"
spec:
  entrypoint: entry
  volumeClaimTemplates:
  - metadata:
      name: workdir
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
  parallelism: 10
  templates:
    - name: entry
      dag:
        tasks:
          - name: get-aws-ips
            template: get-ips
          - name: nmap-aws
            template: run-nmap
            dependencies: [get-aws-ips]
            arguments:
              parameters:
                - name: target
                  value: "{{item}}"
            withParam: "{{tasks.get-aws-ips.outputs.parameters.ips}}"
    - name: get-ips
      script:
        image: byteknight/alpine-curl-jq:latest
        command: [sh, -c]
        args:
          [
            'curl -s ''https://ip-ranges.amazonaws.com/ip-ranges.json'' | jq ''[.prefixes[] | select(.service | contains("EC2")) | .ip_prefix]'' > /mnt/data/ips.json',
          ]
        volumeMounts:
          - name: workdir
            mountPath: /mnt/data
      outputs:
        parameters:
          - name: ips
            valueFrom:
              path: /mnt/data/ips.json
    - name: test-output
      inputs:
        parameters:
          - name: input
      script:
        image: byteknight/alpine-curl-jq
        command: [bash]
        source: |
          echo {{inputs.parameters.input}}
    - name: run-nmap
      inputs:
        parameters:
          - name: target
      container:
        image: securecodebox/nmap:latest
        securityContext:
          privileged: true
          allowPrivilegeEscalation: true
        command: [nmap]
        args:
          [
            "-sV",
            "-vv",
            "-T5",
            "-A",
            "-n",
            "--min-hostgroup",
            "100",
            "--min-parallelism",
            "200",
            "-Pn",
            "-oX",
            "/mnt/data/{{}}.xml",
            "{{inputs.parameters.target}}",
          ]
        volumeMounts:
          - name: workdir
            mountPath: /mnt/data
```

Agora aplicamos no Argo:

```bash
argo submit -n argo nmap-aws.yaml --watch
```

No bash você verá o seguinte resultado:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1672803802868/23b48797-baaa-4957-99e9-8719fcea467b.png align="center")

E assim os pods rodando suas tarefas do Argo Workflow

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1672803919830/22027a29-ed49-41c7-84bc-cb40de67cfc7.png align="center")

Cada pod foi responsável por executar o NMAP pelo menos um ip da lista do range retornado, conforme demonstra o logs do processamento de um pod:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1672804063441/e26ca13c-a0e4-4d95-8bb1-b23809a5fee3.png align="center")

Isso foi um bom exemplo para imaginarmos o que podemos fazer com o poder do Argo Workflow.