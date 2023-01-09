# Configuração de rede Cilium para Kubernetes

Há uma vários e diferentes interfaces de rede de containers (CNI) para Kubernetes. Com tantas opções diferentes, você deve imaginar que existem muitas funcionalidades diferentes.

Alguns são mais para “começar rapidamente” de uma perspectiva de facilidade de uso, e também existem CNIs que são mais focados em segurança ou mais “avançados”.

O Cilium é um CNI construído em torno do eBPF. Ele se concentra fortemente em redes, observabilidade e segurança para redes Kubernetes que usam eBPF para fazer o trabalho.

O que ele faz de uma perspectiva de rede no nível superior não é diferente. Ele permite que você tenha rede em seu cluster Kubernetes, tenha CIDRs específicos para Pods, garanta que os Pods possam se comunicar e todas as outras recursos de rede de um CNI. A maior diferença é que ele faz tudo com um backend eBPF utilizando tabelas de hash em vez de usar kube-proxy e iptables.

### **Deploy Cilium**

Instale o Cilium CLI:

```bash
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/download/v0.12.4/cilium-linux-amd64.tar.gz
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz
```

Execute o seguinte comando para verificar o status do cluster:

```bash
kubectl get nodes
```

Você pode ver que os nodes estão no estado NotReady.

```bash
NAME                   STATUS     ROLES           AGE     VERSION
talos-k8s-master-1     NotReady   control-plane   3m57s   v1.25.4
talos-k8s-master-2     NotReady   <none>          3m22s   v1.25.4
talos-k8s-master-3     NotReady   <none>          2m58s   v1.25.4
talos-k8s-node-20466   NotReady   <none>          3m33s   v1.25.4
talos-k8s-node-46738   NotReady   <none>          3m45s   v1.25.4
talos-k8s-node-62943   NotReady   <none>          3m51s   v1.25.4
```

Isso por porque criamos o cluster Kubernetes sem um CNI, o qual é um plugin de rede.

Veja mais detalhes no link abaixo sobre CNI:

[https://kubernetes.io/pt-br/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/](https://kubernetes.io/pt-br/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)

---

Caso não tenha o HELM instalado, [faça sua instalação com Arkade](https://github.com/alexellis/arkade), uma tool simples e muito útil, sendo considerado uma marketplace open source para desenvolvedores da seguinte forma:

```bash
curl -sLS https://get.arkade.dev | sh && \
arkade get helm && \
mv /root/.arkade/bin/helm /usr/local/bin/
```

---

Execute os seguintes comandos HELM para instalar o Cilium em nosso cluster.

```bash
helm repo add cilium https://helm.cilium.io/ && helm repo update && \
helm upgrade --install cilium cilium/cilium --version 1.12.5 \
   --namespace kube-system \
   --set kubeProxyReplacement=strict \
   --set hostServices.enabled=false \
   --set hostServices.protocols=tcp \
   --set externalIPs.enabled=true \
   --set nodePort.enabled=true \
   --set hostPort.enabled=true \
   --set bpf.masquerade=false \
   --set image.pullPolicy=IfNotPresent \
   --set ipam.mode=kubernetes
```

Você pode ver que o implantamos com a opção `kubeProxyReplacement` definida como `partial`.

Nesse modo, o usuário decide quais componentes para a substituição do eBPF kube-proxy devem ser usados.

Você pode encontrar mais informações sobre os diferentes modos aqui:

[https://docs.cilium.io/en/stable/gettingstarted/kubeproxy-free/#kube-proxy-hybrid-modes](https://docs.cilium.io/en/stable/gettingstarted/kubeproxy-free/#kube-proxy-hybrid-modes)

Execute o seguinte comando para verificar o status do cluster:

```bash
kubectl get nodes
```

Agora você verá os Nodes com o Status Ready:

```bash
NAME                   STATUS     ROLES           AGE   VERSION
talos-k8s-master-1     Ready      control-plane   17m   v1.25.4
talos-k8s-master-2     Ready      <none>          16m   v1.25.4
talos-k8s-master-3     Ready      <none>          16m   v1.25.4
talos-k8s-node-20466   Ready      <none>          16m   v1.25.4
talos-k8s-node-46738   Ready      <none>          17m   v1.25.4
talos-k8s-node-62943   Ready      <none>          17m   v1.25.4
```

Pois existem alguns componentes do Cilium que foram instalados e configurados.

Um Pod em execução em cada node do Kubernetes, implantado por um recurso do Kubernetes denominado `DaemonSet` nos permitiu isso.

Você também pode usar a CLI do Cilium para verificar o status do Cilium:

```bash
cilium status
```

Aqui está a saída esperada:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673146996726/f652cc02-3244-4456-9ea8-3138a51ec2f5.png align="center")

Essa CLI do Cilium é bem limitada, mas tem outra rodando no Pod do Cilium.

E esse tem uma opção `--verbose.`

Execute os seguintes comandos para experimentá-lo:

```bash
pod=$(kubectl -n kube-system get pods -l k8s-app=cilium -o jsonpath='{.items[0].metadata.name}') && \
kubectl -n kube-system exec -q ${pod} -- cilium status --verbose
```

Aqui está a saída esperada:

```bash
KVStore:                Ok   Disabled
Kubernetes:             Ok   1.23 (v1.23.1+k3s1) [linux/amd64]
Kubernetes APIs:        ["cilium/v2::CiliumClusterwideNetworkPolicy", "cilium/v2::CiliumEndpoint", "cilium/v2::CiliumNetworkPolicy", "cilium/v2::CiliumNode", "core/v1::Namespace", "core/v1::Node", "core/v1::Pods", "core/v1::Service", "discovery/v1::EndpointSlice", "networking.k8s.io/v1::NetworkPolicy"]
KubeProxyReplacement:   Partial   [ens4 10.5.0.171]
Host firewall:          Disabled
CNI Chaining:           none
Cilium:                 Ok   1.12.2 (v1.12.2-c7516b9)
NodeMonitor:            Listening for events on 8 CPUs with 64x4096 of shared memory
Cilium health daemon:   Ok   
IPAM:                   IPv4: 5/254 allocated from 10.42.0.0/24, 
Allocated addresses:
  10.42.0.108 (kube-system/local-path-provisioner-9789bdbfb-cbs68)
  10.42.0.174 (router)
  10.42.0.207 (health)
  10.42.0.223 (kube-system/metrics-server-6486d89755-k9s4x)
  10.42.0.246 (kube-system/coredns-84c56f7bfb-n8j8m)
BandwidthManager:       Disabled
Host Routing:           Legacy
Masquerading:           IPTables [IPv4: Enabled, IPv6: Disabled]
Clock Source for BPF:   ktime
Controller Status:      30/30 healthy
  Name                                  Last success   Last error   Count   Message
  cilium-health-ep                      57s ago        never        0       no error   
  dns-garbage-collector-job             6s ago         never        0       no error   
  endpoint-1249-regeneration-recovery   never          never        0       no error   
  endpoint-1300-regeneration-recovery   never          never        0       no error   
  endpoint-1733-regeneration-recovery   never          never        0       no error   
  endpoint-3717-regeneration-recovery   never          never        0       no error   
  endpoint-578-regeneration-recovery    never          never        0       no error   
  endpoint-gc                           6s ago         never        0       no error   
  ipcache-inject-labels                 4m59s ago      5m5s ago     0       no error   
  k8s-heartbeat                         6s ago         never        0       no error   
  link-cache                            13s ago        never        0       no error   
  metricsmap-bpf-prom-sync              6s ago         never        0       no error   
  resolve-identity-1249                 4m58s ago      never        0       no error   
  resolve-identity-1300                 4m58s ago      never        0       no error   
  resolve-identity-1733                 4m57s ago      never        0       no error   
  resolve-identity-3717                 4m57s ago      never        0       no error   
  resolve-identity-578                  4m57s ago      never        0       no error   
  sync-endpoints-and-host-ips           58s ago        never        0       no error   
  sync-lb-maps-with-k8s-services        4m58s ago      never        0       no error   
  sync-policymap-1249                   49s ago        never        0       no error   
  sync-policymap-1300                   48s ago        never        0       no error   
  sync-policymap-1733                   49s ago        never        0       no error   
  sync-policymap-3717                   49s ago        never        0       no error   
  sync-policymap-578                    50s ago        never        0       no error   
  sync-to-k8s-ciliumendpoint (1249)     8s ago         never        0       no error   
  sync-to-k8s-ciliumendpoint (1300)     8s ago         never        0       no error   
  sync-to-k8s-ciliumendpoint (1733)     7s ago         never        0       no error   
  sync-to-k8s-ciliumendpoint (3717)     7s ago         never        0       no error   
  sync-to-k8s-ciliumendpoint (578)      7s ago         never        0       no error   
  template-dir-watcher                  never          never        0       no error   
Proxy Status:            OK, ip 10.42.0.174, 0 redirects active on ports 10000-20000
Global Identity Range:   min 256, max 65535
Hubble:                  Ok   Current/Max Flows: 2169/4095 (52.97%), Flows/s: 7.24   Metrics: Disabled
KubeProxyReplacement Details:
  Status:                 Partial
  Socket LB:              Disabled
  Devices:                ens4 10.5.0.171
  Mode:                   SNAT
  Backend Selection:      Random
  Session Affinity:       Disabled
  Graceful Termination:   Enabled
  NAT46/64 Support:       Disabled
  XDP Acceleration:       Disabled
  Services:
  - ClusterIP:      Enabled
  - NodePort:       Enabled (Range: 30000-32767) 
  - LoadBalancer:   Enabled 
  - externalIPs:    Enabled 
  - HostPort:       Enabled
BPF Maps:   dynamic sizing: on (ratio: 0.002500)
  Name                          Size
  Non-TCP connection tracking   138376
  TCP connection tracking       276752
  Endpoint policy               65535
  Events                        8
  IP cache                      512000
  IP masquerading agent         16384
  IPv4 fragmentation            8192
  IPv4 service                  65536
  IPv6 service                  65536
  IPv4 service backend          65536
  IPv6 service backend          65536
  IPv4 service reverse NAT      65536
  IPv6 service reverse NAT      65536
  Metrics                       1024
  NAT                           276752
  Neighbor table                276752
  Global policy                 16384
  Per endpoint policy           65536
  Session affinity              65536
  Signal                        8
  Sockmap                       65535
  Sock reverse NAT              138376
  Tunnel                        65536
Encryption:            Disabled
Cluster health:        3/3 reachable   (2023-01-08T03:03:17Z)
  Name                 IP              Node        Endpoints
  master (localhost)   10.5.0.171      reachable   reachable
  worker1              10.5.0.169      reachable   reachable
  worker2              10.5.0.172      reachable   reachable
root@master:~#
```

Há muitas informações interessantes aqui, como os detalhes de substituição do kube-proxy.

### `Deploy o Bookinfo app`

Vamos implantar o aplicativo bookinfo para demonstrar vários recursos do Cilium.

Você pode encontrar mais informações sobre este aplicativo aqui.

[https://istio.io/latest/docs/examples/bookinfo/](https://istio.io/latest/docs/examples/bookinfo/)

O diagrama abaixo mostra como os diferentes micros serviços se comunicam.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673147373167/3b003ccc-ae90-4baf-b7f1-2c3d6744b9d5.png align="center")

Execute os seguintes comandos para implantar o aplicativo bookinfo:

```bash
kubectl create namespace bookinfo
kubectl apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/release-1.13/samples/bookinfo/platform/kube/bookinfo.yaml
```

Altere o tipo de serviço da **productpage** para expô-la externamente como tipo `LoadBalancer`:

```bash
kubectl -n bookinfo patch svc productpage -p '{"spec": {"type": "LoadBalancer"}}'
```

### `Load balancing`

Um dos primeiros casos de uso do Cilium é permitir comunicações entre serviços em execução no Kubernetes.

Vamos dar uma olhada no serviço de reviews:

```bash
kubectl -n bookinfo get svc reviews
```

Você deve obter algo assim:

```bash
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
reviews   ClusterIP   10.43.144.23   <none>        9080/TCP   5m18s
```

Agora, vamos dar uma olhada nos pods de reviews:

```bash
kubectl -n bookinfo get pods -l app=reviews -o wide
```

Neste exemplo, você pode ver que uma solicitação indo para o endereço IP 10.43.144.23 deve ter carga balanceada nos seguintes endereços IP:

10.42.0.176 10.42.0.4 10.42.0.154 Isso geralmente é tratado pelo componente kube-proxy, mas quando o Cilium é implantado, ele é gerenciado pelo Cilium usando eBPF.

Você pode dar uma olhada na configuração de balanceamento de carga do Cílio usando os seguintes comandos:

```bash
pod=$(kubectl -n kube-system get pods -l k8s-app=cilium -o jsonpath='{.items[0].metadata.name}') && \
kubectl -n kube-system exec -q ${pod} -- cilium bpf lb list
```

A saída esperada:

```bash
SERVICE ADDRESS      BACKEND ADDRESS (REVNAT_ID) (SLOT)
10.43.144.23:9080    10.42.0.231:9080 (8) (1)                      
                     10.42.0.252:9080 (8) (2)                      
                     10.42.2.57:9080 (8) (3)                       
                     0.0.0.0:0 (8) (0) [ClusterIP, non-routable]   
10.43.234.55:443     10.5.0.171:4244 (5) (1)                       
                     10.5.0.169:4244 (5) (2)                       
                     0.0.0.0:0 (5) (0) [ClusterIP, non-routable]   
                     10.5.0.172:4244 (5) (3)                       
10.43.151.156:443    0.0.0.0:0 (4) (0) [ClusterIP, non-routable]   
                     10.42.0.223:443 (4) (1)                       
10.43.5.206:9080     10.42.0.21:9080 (9) (1)                       
                     0.0.0.0:0 (9) (0) [ClusterIP, non-routable]   
10.5.0.172:9080      0.0.0.0:0 (15) (0) [LoadBalancer]             
                     10.42.0.21:9080 (15) (1)                      
10.43.0.10:9153      0.0.0.0:0 (3) (0) [ClusterIP, non-routable]   
                     10.42.0.246:9153 (3) (1)                      
10.43.0.1:443        0.0.0.0:0 (1) (0) [ClusterIP, non-routable]   
                     10.5.0.171:6443 (1) (1)                       
10.5.0.169:9080      0.0.0.0:0 (14) (0) [LoadBalancer]             
                     10.42.0.21:9080 (14) (1)                      
0.0.0.0:9080         0.0.0.0:0 (13) (0) [HostPort, non-routable]   
                     10.42.0.235:9080 (13) (1)                     
10.43.0.10:53        10.42.0.246:53 (2) (1)                        
                     0.0.0.0:0 (2) (0) [ClusterIP, non-routable]   
10.43.234.98:9080    10.42.0.116:9080 (7) (1)                      
                     0.0.0.0:0 (7) (0) [ClusterIP, non-routable]   
10.5.0.171:9080      0.0.0.0:0 (12) (0) [LoadBalancer]             
                     10.42.0.21:9080 (12) (1)                      
10.5.0.171:30894     0.0.0.0:0 (10) (0) [NodePort]                 
                     10.42.0.21:9080 (10) (1)                      
10.43.124.225:9080   0.0.0.0:0 (6) (0) [ClusterIP, non-routable]   
                     10.42.0.220:9080 (6) (1)                      
0.0.0.0:30894        10.42.0.21:9080 (11) (1)                      
                     0.0.0.0:0 (11) (0) [NodePort, non-routable]
```

A parte interessante do serviço de reviews é esta:

```bash
10.43.144.23:9080    10.42.0.231:9080 (8) (1)                      
                     10.42.0.252:9080 (8) (2)                      
                     10.42.2.57:9080 (8) (3)                       
                     0.0.0.0:0 (8) (0) [ClusterIP, non-routable]
```

Você pode ver os endereços IP correspondentes ao serviço e aos pods.

Vamos também dar uma olhada em como as comunicações entre os nodes do Kubernetes são tratadas.

```bash
pod=$(kubectl -n kube-system get pods -l k8s-app=cilium -o jsonpath='{.items[0].metadata.name}') && \
kubectl -n kube-system exec -q ${pod} -- cilium bpf tunnel list
```

Podemos listar os túneis criados pelo Cilium:

```bash
TUNNEL        VALUE
10.42.2.0:0   10.5.0.169:0   
10.42.1.0:0   10.5.0.172:0
```

Você pode ver que executamos o comando do Pod em execução no node master do Kubernetes porque os túneis listados são os que permitem acessar os Pods em execução nos outros dois nodes.

### `Conceitos`

O Cilium cria automaticamente um endpoint para cada Pod.

Antes de mais nada, vamos dimensionar a implantação de details-v1 para obter mais 2 pods:

```bash
kubectl -n bookinfo scale deploy/details-v1 --replicas=2
```

Você pode listar os endpoints dos pods bookinfo usando o seguinte comando:

```bash
kubectl -n bookinfo get ciliumendpoints
```

Você deve obter algo como isto:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673149057685/1b579c14-40be-4917-80d8-dc426145f054.png align="center")

Você pode ver que **cada endpoint tem um ID de endpoint exclusivo** e que 2 pods da **mesma implantação compartilham o mesmo ID de identidade** (porque contém as mesmas labels).

Observe que você pode ignorar os que começam com **svclb**.

Eles correspondem a Pods criados para tornar o serviço productpage acessível.

Agora, vamos dimensionar o deployment de details-v1 para 1 réplica:

```bash
kubectl -n bookinfo scale deploy/details-v1 --replicas=1
```

Vamos dar uma olhada no **endpoint productpage**.

```bash
kubectl -n bookinfo get ciliumendpoints -l app=productpage -o yaml
```

x

```bash
apiVersion: v1
items:
- apiVersion: cilium.io/v2
  kind: CiliumEndpoint
  metadata:
    creationTimestamp: "2023-01-08T03:17:22Z"
    generation: 1
    labels:
      app: productpage
      pod-template-hash: 5586c4d4ff
      version: v1
    name: productpage-v1-5586c4d4ff-npzcr
    namespace: bookinfo
    ownerReferences:
    - apiVersion: v1
      kind: Pod
      name: productpage-v1-5586c4d4ff-npzcr
      uid: d1b4c1c1-d291-4c6d-be82-1394c4714bf1
    resourceVersion: "1584"
    uid: 2fd574d8-ddbf-479f-a70b-24d39f48e811
  status:
    encryption: {}
    external-identifiers:
      container-id: 1e4101a4e7b13557188962f04fae6d2724ce8d5e2ff5a50732120dc119112412
      k8s-namespace: bookinfo
      k8s-pod-name: productpage-v1-5586c4d4ff-npzcr
      pod-name: bookinfo/productpage-v1-5586c4d4ff-npzcr
    id: 207
    identity:
      id: 23407
      labels:
      - k8s:app=productpage
      - k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo
      - k8s:io.cilium.k8s.policy.cluster=default
      - k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-productpage
      - k8s:io.kubernetes.pod.namespace=bookinfo
      - k8s:version=v1
    networking:
      addressing:
      - ipv4: 10.42.0.21
      node: 10.5.0.171
    policy:
      egress:
        enforcing: false
        state: <status disabled>
      ingress:
        enforcing: false
        state: <status disabled>
    state: ready
    visibility-policy-status: <status disabled>
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

A parte mais interessante é a seção `identity`:

```bash
id: 207
    identity:
      id: 23407
      labels:
      - k8s:app=productpage
      - k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo
      - k8s:io.cilium.k8s.policy.cluster=default
      - k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-productpage
      - k8s:io.kubernetes.pod.namespace=bookinfo
      - k8s:version=v1
```

Você pode ver o ID da `identity`, mas também todas labels derivadas do Kubernetes.

Usaremos essas labels mais tarde quando usarmos as políticas de rede do Cilium.

Por serem recursos personalizados, você também pode listar todas as `identity` usando o seguinte comando:

```bash
kubectl get ciliumidentities
```

Você deve obter algo assim:

```bash
NAME    NAMESPACE     AGE
53694   kube-system   44m
7200    kube-system   44m
37245   kube-system   44m
22047   bookinfo      26m
33566   bookinfo      26m
4752    bookinfo      26m
43116   bookinfo      26m
15745   bookinfo      26m
23407   bookinfo      26m
22101   bookinfo      25m
```

E você pode ver quais labels correspondem a uma `identity` específica:

```bash
kubectl get ciliumidentity 23407 -o yaml
```

Retornará:

```bash
apiVersion: cilium.io/v2
kind: CiliumIdentity
metadata:
  creationTimestamp: "2023-01-08T03:17:22Z"
  generation: 1
  labels:
    app: productpage
    io.cilium.k8s.policy.cluster: default
    io.cilium.k8s.policy.serviceaccount: bookinfo-productpage
    io.kubernetes.pod.namespace: bookinfo
    version: v1
  name: "23407"
  resourceVersion: "1583"
  uid: 10d33c42-60ed-4bb2-a338-a9f0691441c9
security-labels:
  k8s:app: productpage
  k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name: bookinfo
  k8s:io.cilium.k8s.policy.cluster: default
  k8s:io.cilium.k8s.policy.serviceaccount: bookinfo-productpage
  k8s:io.kubernetes.pod.namespace: bookinfo
```

### `Observabilidade`

Benefícios do **eBPF** e do **Cilium** é sua capacidade de fornecer métricas sobre todas as comunicações de rede que ocorrem no cluster.

O **Hubble** é construído sobre o Cilium e o eBPF para permitir uma visibilidade profunda da comunicação e do comportamento dos serviços, bem como da infraestrutura de rede de maneira totalmente transparente.

Você também pode configurar o **Cilium** e o **Hubble** para atender às métricas do Prometheus.

Vamos atualizar o **Deployment do Cilium para habilitar o Hubble** e as métricas do **Prometheus** do servidor.

```bash
helm upgrade --install cilium cilium/cilium --version 1.12.2 \
   --namespace kube-system \
   --set hubble.enabled=true \
   --set hubble.metrics.enabled="{dns,drop,tcp,flow,icmp,http}" \
   --set hubble.relay.enabled=true \
   --set hubble.ui.enabled=true \
   --set prometheus.enabled=true \
   --set operator.prometheus.enabled=true \
   --set kubeProxyReplacement=partial \
   --set hostServices.enabled=false \
   --set hostServices.protocols=tcp \
   --set externalIPs.enabled=true \
   --set nodePort.enabled=true \
   --set hostPort.enabled=true \
   --set bpf.masquerade=false \
   --set image.pullPolicy=IfNotPresent \
   --set ipam.mode=kubernetes
```

Altere o tipo de `Service` para `LoadBalancer` para expor a IU externamente:

```bash
kubectl -n kube-system patch svc hubble-ui -p '{"spec": {"type": "LoadBalancer"}}'
```

Execute o seguinte comando para gerar algum tráfego:

```bash
for i in {1..25}; do
   curl -s -o /dev/null -w "%{http_code}" http://$(kubectl -n bookinfo get svc productpage -o jsonpath='{.status.loadBalancer.ingress[0].*}'):9080/productpage
   printf "\n"
   sleep 1
done
```

Acesse **Hubble UI** e selecione o namespace **bookinfo**.

Ele exibirá um gráfico mostrando todas as comunicações que ocorrem entre os serviços.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673150078501/8cc6b603-1a6a-417b-b1e9-2cf050cf9c67.png align="center")

### `Como funciona nos bastidores?`

O componente de servidor do Hubble está encapsulado no agente do CIlium.

O agente Cilium em execução em cada node do Kubernetes expõe as métricas do Hubble por meio do gRPC.

O componente de retransmissão do Hubble está ciente de todas as instâncias do Cílio e consolida todas as métricas.

Vamos executar o seguinte comando para ver os dados expostos pelo Relay do Hubble:

```bash
pod=$(kubectl -n kube-system get pods -l k8s-app=hubble-relay -o jsonpath='{.items[0].metadata.name}') && \
kubectl -n kube-system debug -i -q ${pod} --image=fullstorydev/grpcurl -- grpcurl -plaintext localhost:4245 observer.Observer/GetFlows
```

Agora, vamos implantar o Prometheus e o Grafana para dar uma olhada nas métricas atendidas pelo Cilium e pelo Hubble:

```bash
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.11/examples/kubernetes/addons/prometheus/monitoring-example.yaml
```

Altere o tipo de serviço para expor a IU do Grafana externamente:

```bash
kubectl -n cilium-monitoring patch svc grafana -p '{"spec": {"type": "LoadBalancer"}}'
```

Acesse a guia Grafana e dê uma olhada nos painéis.

### `Segurança L3/L4`

Outro benefício do eBPF e do Cilium é sua capacidade de proteger as comunicações serviço a serviço.

O objetivo deste laboratório não é examinar os diferentes casos de uso das políticas de rede L3/L4, mas demonstrar como validar se as políticas são aplicadas corretamente.

A documentação do Cilium fornece muitos exemplos mostrando como criar políticas de rede L3/L4 para vários casos de uso.

[https://docs.cilium.io/en/stable/policy/language/#layer-3-examples](https://docs.cilium.io/en/stable/policy/language/#layer-3-examples)

---

Digamos que você queira definir explicitamente quais serviços podem se comunicar juntos. Você pode usar políticas de rede L3 para essa finalidade.

Mas aqui, queremos apenas permitir uma comunicação para uma porta específica. Portanto, precisamos usar uma política de rede L4.

No contexto do aplicativo Bookinfo, queremos que o serviço productpage possa se comunicar com o serviço de reviews na porta 9080.

Vamos criar a política de rede L4 correspondente.

```bash
cat << EOF | kubectl -n bookinfo apply -f -
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: reviews
  namespace: bookinfo
spec:
  description: "Allow the productpage service to communicate with the reviews service"
  endpointSelector:
    matchLabels:
      app: reviews
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: productpage
    toPorts:
    - ports:
      - port: "9080"
        protocol: TCP
EOF
```

Observe que não prefixamos a chave do label com `k8s`. Nesse caso, é equivalente a `any`: que corresponderá a todas labels independentemente de sua origem.

Se você atualizar a página da Web na guia Bookinfo, verá que o aplicativo ainda está funcionando.

Mas agora, se você tentar enviar uma solicitação do serviço de detalhes para o serviço de reviews, ela falhará.

Vamos checar.

```bash
pod=$(kubectl -n bookinfo get pods -l app=details -o jsonpath='{.items[0].metadata.name}') && \
kubectl -n bookinfo debug -i -q ${pod} --image=curlimages/curl -- curl -s -o /dev/null -w "%{http_code}" http://reviews:9080/reviews/0
```

Deve confirmar que a comunicação não é permitida.

Você também pode executar os seguintes comandos no segundo terminal para exibir os pacotes descartados:

```bash
node=$(kubectl -n bookinfo get pods -l app=reviews -o jsonpath='{.items[0].spec.nodeName}') && \
pod=$(kubectl -n kube-system get pods -l k8s-app=cilium -o json | jq -r ".items[] | select(.spec.nodeName==\"${node}\") | .metadata.name" | tail -1) && \
kubectl -n kube-system exec -q $pod -- cilium monitor --type drop
```

Aqui está a saída esperada:

```bash
Listening for events on 8 CPUs with 64x4096 of shared memory
Press Ctrl-C to quit
xx drop (Policy denied) flow 0xe07b899c to endpoint 148, file bpf_lxc.c line 1916, , identity 22047->15745: 10.42.0.220:56148 -> 10.42.0.231:9080 tcp SYN
level=info msg="Initializing dissection cache..." subsys=monitor
xx drop (Unsupported L3 protocol) flow 0x0 to endpoint 0, file bpf_lxc.c line 1368, , identity 41658->unknown: fe80::7495:4cff:fe41:9d5a -> ff02::2 RouterSolicitation
xx drop (Unsupported L3 protocol) flow 0x0 to endpoint 0, file bpf_lxc.c line 1368, , identity 9415->unknown: fe80::f050:1aff:fe75:cf06 -> ff02::2 RouterSolicitation
xx drop (Unsupported L3 protocol) flow 0x0 to endpoint 0, file bpf_lxc.c line 1368, , identity 3436->unknown: fe80::48f3:55ff:fe08:c386 -> ff02::2 RouterSolicitation
```

Você pode ver que as quedas estão acontecendo quando a `identity` **22047** se comunica com a `identity` **15745** em nosso exemplo.

Você pode executar o seguinte comando para obter mais informações sobre essas `identity`.

Não se esqueça de substituir o número de `identity` pelo que você obteve na saída anterior.

```bash
pod=$(kubectl -n kube-system get pods -l k8s-app=cilium -o jsonpath='{.items[0].metadata.name}') && \
kubectl -n kube-system exec -q ${pod} -- cilium identity get 22047
```

Aqui está a saída esperada:

```bash
ID      LABELS
22047   k8s:app=details
        k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo
        k8s:io.cilium.k8s.policy.cluster=default
        k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-details
        k8s:io.kubernetes.pod.namespace=bookinfo
        k8s:version=v1
```

Observe que o Cilium também está criando alguns recursos personalizados do Kubernetes, como `CiliumIdentities` e `CiliumEndpoints`. Ele permite que os usuários do Kubernetes entendam quais recursos estão disponíveis.

Neste laboratório, demonstramos como o Cilium pode impor políticas L4, mas a experiência do usuário não é boa.

Em vez de não obter nenhuma resposta, um usuário prefere obter um código de resposta 403 padrão para entender que a comunicação não funciona porque não é permitida.

**Este é o propósito das políticas L7.**

### `Segurança L7`

O objetivo deste laboratório não é examinar os diferentes casos de uso das políticas de rede L7, mas demonstrar como elas são aplicadas nos bastidores.

A documentação do Cilium fornece muitos exemplos mostrando como criar políticas de rede L7 para vários casos de uso.

[https://docs.cilium.io/en/stable/policy/language/#layer-7-examples](https://docs.cilium.io/en/stable/policy/language/#layer-7-examples)

---

Vamos ver como as políticas L7 podem ser usadas para aprimorar a experiência do usuário.

Mas antes de atualizarmos a política, vamos dar uma olhada nos processos que estão sendo executados no(s) pod(s) Cilium onde os pods de reviews estão sendo executados.

```bash
kubectl -n bookinfo get pods -l app=reviews -o json | jq -r '.items[].spec.nodeName' | sort -u | while read node; do
  kubectl -n kube-system get pods -l k8s-app=cilium -o json | jq -r ".items[] | select(.spec.nodeName==\"${node}\") | .metadata.name" | while read pod; do
    kubectl -n kube-system exec -q ${pod} -- ps -ef
  done
done
```

A saída deve ficar assim:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673151143684/6a1d5214-8713-477c-b6b5-0994c29aca3e.png align="center")

Se os pods de reviews estiverem em execução em diferentes nodes do Kubernetes, você obterá essa saída várias vezes, como demostrado acima.

Como você pode ver, existem apenas 2 processos (**cilium-agent e cilium-health-responder**) em execução no container.

Qual é o ponto?

**Vamos atualizar a política de rede (de L4 para L7):**

```bash
cat << EOF | kubectl -n bookinfo apply -f -
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: reviews
  namespace: bookinfo
spec:
  description: "Allow the productpage service to communicate with the reviews service"
  endpointSelector:
    matchLabels:
      app: reviews
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: productpage
    toPorts:
    - ports:
      - port: "9080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
  - toPorts:
    - ports:
      - port: "9080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/nopathallowed"
EOF
```

Atualize a página da Web na guia Bookinfo. Você verá que o aplicativo ainda está funcionando.

Mas, vamos dar uma olhada nos processos em execução agora:

```bash
kubectl -n bookinfo get pods -l app=reviews -o json | jq -r '.items[].spec.nodeName' | sort -u | while read node; do
  kubectl -n kube-system get pods -l k8s-app=cilium -o json | jq -r ".items[] | select(.spec.nodeName==\"${node}\") | .metadata.name" | while read pod; do
    kubectl -n kube-system exec -q ${pod} -- ps -ef
  done
done
```

O retorno será:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673151449108/ef3cd7bf-3a6a-41bb-92f7-7f925110ea10.png align="center")

**Por que há um processo cilium-envoy em execução agora?**

A razão é simples. O **Envoy** é necessário para aplicar as **políticas L7.** E eles são aplicados no node do Kubernetes em que os serviços de destino estão sendo executados.

Vamos verificar qual é a experiência do usuário agora quando a comunicação não é permitida:

```bash
pod=$(kubectl -n bookinfo get pods -l app=details -o jsonpath='{.items[0].metadata.name}') && \
kubectl -n bookinfo debug -i -q ${pod} --image=curlimages/curl -- curl -s -o /dev/null -w "%{http_code}" http://reviews:9080/reviews/0;echo  ''
```

Deve confirmar que a comunicação não é permitida, mas a experiência do usuário é melhor agora. O usuário recebe um código de resposta 403 explícito.

Observe que tivemos que criar uma regra falsa para as solicitações vindas de todos os outros Pods, exceto o da página do produto.

**É porque o Cilium suporta políticas de negação para L7**

### `Vamos nos aprofundar em como o Cilium configura o Envoy para aplicar as políticas L7.`

A instância do Envoy implantada pelo Cilium inclui um filtro `HTTP cilium.l7policy` que obtém seus dados por meio de um `endpoint xds local`.

Você pode ver os dados fornecidos pelo `endpoint xds` executando o seguinte snippet:

```bash
node=$(kubectl -n bookinfo get pods -l app=reviews -o jsonpath='{.items[0].spec.nodeName}')
pod=$(kubectl -n kube-system get pods -l k8s-app=cilium -o json | jq -r ".items[] | select(.spec.nodeName==\"${node}\") | .metadata.name" | tail -1)
kubectl -n kube-system exec -q $pod -- apt update
kubectl -n kube-system exec -q $pod -- apt -y install curl
kubectl -n kube-system exec -q $pod -- curl -k -L https://github.com/fullstorydev/grpcurl/releases/download/v1.8.6/grpcurl_1.8.6_linux_x86_64.tar.gz --output /tmp/grpcurl.tar.gz
kubectl -n kube-system exec -q $pod -- bash -c "cd /tmp && tar zxvf /tmp/grpcurl.tar.gz"
while true; do
cat <<EOF | kubectl -n kube-system exec -q -i $pod -- /tmp/grpcurl -d @ -plaintext -unix /var/run/cilium/xds.sock cilium.NetworkPolicyDiscoveryService/StreamNetworkPolicies > /tmp/output
{
  "node": $(kubectl -n kube-system exec -q $pod -- curl -s --unix-socket /var/run/cilium/envoy-admin.sock http://localhost/config_dump | jq -r ".configs[0].bootstrap.node"),
  "resourceNames": [
  ],
  "typeUrl": "type.googleapis.com/cilium.NetworkPolicy"
}
EOF
if [ -s /tmp/output ]; then
  cat /tmp/output
  break
fi
done
```

Pode levar algum tempo até você obter a resposta.

A saída deve ficar assim:

xxx

Você pode ver todas as políticas que serão aplicadas pelo Envoy (e os endpoints Cilium correspondentes).

A criação de uma política de rede L7 também aumenta a visibilidade das comunicações correspondentes.

Execute os seguintes comandos para observar os fluxos L7:

```bash
node=$(kubectl -n bookinfo get pods -l app=reviews -o jsonpath='{.items[0].spec.nodeName}') && \
pod=$(kubectl -n kube-system get pods -l k8s-app=cilium -o json | jq -r ".items[] | select(.spec.nodeName==\"${node}\") | .metadata.name" | tail -1) && \
kubectl -n kube-system exec -q $pod -- cilium monitor -t l7
```

Em seguida, atualize a guia Bookinfo várias vezes para gerar algum tráfego.

A saída do comando agora deve se parecer com o seguinte:

```bash
Listening for events on 8 CPUs with 64x4096 of shared memory
Press Ctrl-C to quit
<- Request http from 207 ([k8s:io.kubernetes.pod.namespace=bookinfo k8s:version=v1 k8s:app=productpage k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-productpage]) to 148 ([k8s:app=reviews k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-reviews k8s:io.kubernetes.pod.namespace=bookinfo k8s:version=v3]), identity 23407->15745, verdict Forwarded GET http://reviews:9080/reviews/0 => 0
<- Response http to 207 ([k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-productpage k8s:io.kubernetes.pod.namespace=bookinfo k8s:version=v1 k8s:app=productpage k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo]) from 148 ([k8s:app=reviews k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-reviews k8s:io.kubernetes.pod.namespace=bookinfo k8s:version=v3]), identity 15745->23407, verdict Forwarded GET http://reviews:9080/reviews/0 => 200
```

Observe que você também pode obter visibilidade do protocolo L7 anotando pods.

Vamos fazer isso para o Pod de detalhes:

```bash
pod=$(kubectl -n bookinfo get pods -l app=details -o jsonpath='{.items[0].metadata.name}') && \
kubectl -n bookinfo annotate pod ${pod} io.cilium.proxy-visibility="<Ingress/9080/TCP/HTTP>"
```

Execute os seguintes comandos para observar os fluxos L7:

```bash
node=$(kubectl -n bookinfo get pods -l app=details -o jsonpath='{.items[0].spec.nodeName}') && \
pod=$(kubectl -n kube-system get pods -l k8s-app=cilium -o json | jq -r ".items[] | select(.spec.nodeName==\"${node}\") | .metadata.name" | tail -1) && \
kubectl -n kube-system exec -q $pod -- cilium monitor -t l7
```

Em seguida, atualize a guia Bookinfo várias vezes para gerar algum tráfego.

A saída agora deve ter detalhes sobre as solicitações que vão do serviço productpage para o serviço de detalhes.

```bash
Listening for events on 8 CPUs with 64x4096 of shared memory
Press Ctrl-C to quit
<- Request http from 207 ([k8s:app=productpage k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-productpage k8s:io.kubernetes.pod.namespace=bookinfo k8s:version=v1]) to 203 ([k8s:version=v1 k8s:app=details k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-details k8s:io.kubernetes.pod.namespace=bookinfo]), identity 23407->22047, verdict Forwarded GET http://details:9080/details/0 => 0
<- Response http to 207 ([k8s:app=productpage k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-productpage k8s:io.kubernetes.pod.namespace=bookinfo k8s:version=v1]) from 203 ([k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-details k8s:io.kubernetes.pod.namespace=bookinfo k8s:version=v1 k8s:app=details]), identity 22047->23407, verdict Forwarded GET http://details:9080/details/0 => 200
<- Request http from 207 ([k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-productpage k8s:io.kubernetes.pod.namespace=bookinfo k8s:version=v1 k8s:app=productpage k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo]) to 203 ([k8s:io.kubernetes.pod.namespace=bookinfo k8s:version=v1 k8s:app=details k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-details]), identity 23407->22047, verdict Forwarded GET http://details:9080/details/0 => 0
<- Response http to 207 ([k8s:io.kubernetes.pod.namespace=bookinfo k8s:version=v1 k8s:app=productpage k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-productpage]) from 203 ([k8s:app=details k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-details k8s:io.kubernetes.pod.namespace=bookinfo k8s:version=v1]), identity 22047->23407, verdict Forwarded GET http://details:9080/details/0 => 200
<- Request http from 207 ([k8s:version=v1 k8s:app=productpage k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-productpage k8s:io.kubernetes.pod.namespace=bookinfo]) to 203 ([k8s:io.kubernetes.pod.namespace=bookinfo k8s:version=v1 k8s:app=details k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-details]), identity 23407->22047, verdict Forwarded GET http://details:9080/details/0 => 0
<- Response http to 207 ([k8s:io.kubernetes.pod.namespace=bookinfo k8s:version=v1 k8s:app=productpage k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo k8s:io.cilium.k8s.policy.cluster=default k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-productpage]) from 203 ([k8s:io.cilium.k8s.policy.serviceaccount=bookinfo-details k8s:io.kubernetes.pod.namespace=bookinfo k8s:version=v1 k8s:app=details k8s:io.cilium.k8s.namespace.labels.kubernetes.io/metadata.name=bookinfo k8s:io.cilium.k8s.policy.cluster=default]), identity 22047->23407, verdict Forwarded GET http://details:9080/details/0 => 200
```