# eBPF com BumbleBee

Usado para estender com segurança e eficiência os recursos do kernel sem a necessidade de alterar o código fonte do kernel ou carregar os módulos do kernel.

**eBPF** é uma tecnologia revolucionária com origem no **kernel do Linux** que pode executar programas em área restrita em um kernel do sistema operacional. Ele é usado para estender com segurança e eficiência os recursos do kernel sem a necessidade de alterar o código fonte do kernel ou carregar os módulos do kernel.

Sistema operacional sempre foi um local ideal para implementar observabilidade, segurança e funcionalidade de rede devido à capacidade privilegiada do kernel de supervisionar e controlar todo o sistema. Ao mesmo tempo, o kernel de um sistema operacional é difícil de evoluir devido ao seu papel central e alta exigência de estabilidade e segurança.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673046932173/ca016b2a-1d3e-4f50-879b-395d4bbe9d05.png align="center")

Como você pode ver no diagrama, você precisa de um programa **BPF** em execução no **Kernel** e um aplicativo de espaço do usuário em execução no **espaço do usuário** que interaja com o programa BPF (para coletar dados, por exemplo).

### `Desenvolvendo eBPF com BumbleBee`

Construir, executar e distribuir programas eBPF através de imagens do Docker.

Permite que você se concentre em codificar o código eBPF, enquanto cuida dos componentes do espaço do usuário, assim expondo automaticamente seus dados como métricas ou logs.

Instale o bee CLI em plataforma Linux, pois se você usa Mac, precisará de um ambiente virtualizado para isso:

```bash
curl -sL https://run.solo.io/bee/install | sh
```

Com a sequência dos comandos acima, você terá o seguinte retorno.

```bash
<string>:1: DeprecationWarning: The distutils package is deprecated and slated for removal in Python 3.12. Use setuptools or check PEP 632 for potential alternatives
Attempting to download bee version v0.0.14
Downloading bee-linux-amd64...
Download complete!, validating checksum...
Checksum valid.
bee was successfully installed 🎉

Add the bumblebee CLI to your path with:
  export PATH=$HOME/.bumblebee/bin:$PATH

Now run:
  bee init     # Initialize simple eBPF program to run with bee
Please see visit the bumblebee website for more info:  https://github.com/solo-io/bumblebee
```

Agora execute o export para acessar o executável do bee:

```bash
export PATH=$HOME/.bumblebee/bin:$PATH
```

Agora vamos criar um esqueleto para nosso programa eBPF:

```bash
bee init
```

A primeira opção com a qual você será confrontado é a linguagem com a qual desenvolverá seu probe.

Atualmente, apenas C é suportado, mas o suporte para Rust também está planejado.

```bash
Use the arrow keys to navigate: ↓ ↑ → ←
? What language do you wish to use for the filter:
  ▸ C
```

Agora que selecionamos o idioma a ser usado, será solicitado que você selecione o tipo de programa que deseja criar. Como o eBPF permite que você escreva programas que podem se conectar a praticamente qualquer funcionalidade do kernel, existem vários "tipos" de programas que você pode criar.

O bee atualmente tem dois pontos de partida: programas baseados em rede ou sistema de arquivos. Os programas de rede serão focados em conectar-se a várias funções da rede do kernel, enquanto os programas de sistema de arquivos se conectam a operações de arquivo, como chamadas `open()`. Para este tutorial, vamos selecionar "Rede".

```bash
 INFO  Selected Language: C
Use the arrow keys to navigate: ↓ ↑ → ←
? What type of program to initialize:
  ▸ Network
    FileSystem
```

Em seguida, você será solicitado a fornecer o tipo de mapa global que gostaria de usar. Os **HashMaps** são o instrumento através do qual o espaço do usuário eBPF e os programas de kernel podem se comunicar entre si.

Informações mais detalhadas sobre esses `HashMap`, bem como os diferentes tipos de mapas disponíveis, podem ser encontradas na seção de HashMaps eBPF da documentação do BPF linux. **Vamos usar o HashMap.**

```bash
 INFO  Selected Language: C
 INFO  Selected Program Type: Network
Use the arrow keys to navigate: ↓ ↑ → ←
? What type of map should we initialize:
    RingBuffer
  ▸ HashMap
```

Depois de decidir sobre um tipo de HashMap, você será solicitado a decidir sobre um formato de saída.

Normalmente, o desenvolvimento de aplicativos eBPF requer a gravação de espaço do usuário e código alocado no kernel.

No entanto, com o bee, você só precisa desenvolver o código no kernel e, em seguida, o bee pode manipular e gerar automaticamente os dados de seus HashMaps eBPF.

Além disso, o bee pode emitir métricas dos dados recebidos por seus HashMaps eBPF. Dependendo do seu caso de uso, você pode simplesmente exibir os dados em seu HashMap como texto, o que corresponde ao tipo de saída de impressão.

No entanto, se você quiser gerar métricas a partir dos dados, poderá selecionar um tipo de métrica. Atualmente, as métricas do tipo contador e medidor são suportadas. Por enquanto, escolheremos `print`, o que, novamente, produzirá apenas os dados do mapa como texto e não emitirá nenhuma métrica.

```bash
 INFO  Selected Language: C
 INFO  Selected Program Type: Network
 INFO  Selected Map Type: HashMap
Use the arrow keys to navigate: ↓ ↑ → ←
? What type of output would you like from your map:
  ▸ print
    counter
    gauge
```

Finalmente, decidiremos sobre a localização do arquivo.

```bash
✔ BPF Program File Location: probe.c
```

O arquivo criado com a nomenclatura `probe.c` deve ter o seguinte código em C:

```c
#include "vmlinux.h"
#include "bpf/bpf_helpers.h"
#include "bpf/bpf_core_read.h"
#include "bpf/bpf_tracing.h"
#include "solo_types.h"

// 1. Change the license if necessary
char __license[] SEC("license") = "Dual MIT/GPL";

struct dimensions_t {
	// 2. Add dimensions to your value. This struct will be used as the key in the hash map of your data.
	// These will be treated as labels on your metrics.
	// In this example we will have single field which contains the PID of the process
	u32 pid;
} __attribute__((packed));

// This is the definition for the global map which both our
// bpf program and user space program can access.
// More info and map types can be found here: https://www.man7.org/linux/man-pages/man2/bpf.2.html
struct {
	__uint(max_entries, 1 << 24);
	__uint(type, BPF_MAP_TYPE_HASH);
	__type(key, struct dimensions_t);
	__type(value, u64);
} values SEC(".maps.print");

SEC("kprobe/tcp_v4_connect")
int BPF_KPROBE(tcp_v4_connect, struct sock *sk)
{
	// initialize our struct which will be the key in the hash map
	struct dimensions_t key;
	// initialize variable used to track PID of process calling tcp_v4_connect
	u32 pid;
	// define variable used to track the count of function calls, and a pointer to it for plumbing
	u64 counter;
	u64 *counterp;

	// get the pid for the current process which has entered the tcp_v4_connect function
	pid = bpf_get_current_pid_tgid();
	key.pid = pid;

	// check if we have an existing value for this key
	counterp = bpf_map_lookup_elem(&values, &key);
	if (!counterp) {
		// debug log to help see how the program works
		bpf_printk("no entry found for pid: %u}", key.pid);
		// no entry found, so this is the first occurrence, set value to 1
		counter = 1;
	}
	else {
		bpf_printk("found existing value '%llu' for pid: %u", *counterp, key.pid);
		// we found an entry, so let's increment the existing value for this PID
		counter = *counterp + 1;
	}
	// update our map with the new value of the counter
	bpf_map_update_elem(&values, &key, &counter, 0);


	return 0;
}
```

Faça o download na `libbpf-tools`, o módulo `tcpconnect.c`

```bash
curl https://raw.githubusercontent.com/iovisor/bcc/master/libbpf-tools/tcpconnect.c -o tcpconnect.c
```

Use o bee cli para compilar seu programa e armazená-lo como uma imagem Docker:

```bash
bee build tcpconnect.c ttl.sh/ebpf-solo-tcpconnect:1h
```

Você tera o retorno.

```bash
/home/ubuntu # bee build probe.c ttl.sh/ebpf-solo-tcpconnect:1h                                                                                                                                
 SUCCESS  Successfully compiled "tcpconnect.c" and wrote it to "tcpconnect.o"                                                                                                                                                                          
 SUCCESS  Saved BPF OCI image to ttl.sh/ebpf-solo-tcpconnect:1h  
```

Envie a imagem para o serviço do Registry Efêmero, denominado [ttl.sh](https://ttl.sh/)

```bash
bee push ttl.sh/ebpf-solo-tcpconnect:1h
```

Você pode enviar e receber os programas BumbleBee de/para qualquer registro compatível com Docker (como o Registry privado ou público do Docker em execução nesta máquina).

Você terá o retorno.

```bash
 SUCCESS  Pushing image ttl.sh/ebpf-solo-tcpconnect:1h to remote registry
```

Finalmente, você pode executar seu programa:

```bash
bee run ttl.sh/ebpf-solo-tcpconnect:1h
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673127938461/e8b42cd6-ebb9-4049-af7e-6aa99bd75bb7.png align="center")

Você pode executar qualquer comando curl no segundo terminal para ver as novas entradas exibidas na interface do usuário do BumbleBee.

Agora, execute o seguinte comando em uma guia diferente enquanto o programa estiver em execução:

```bash
curl http://localhost:9091/metrics
```

```bash
# HELP ebpf_solo_io_events_hash 
# TYPE ebpf_solo_io_events_hash counter
ebpf_solo_io_events_hash{daddr="10.42.0.4",saddr="10.42.0.1"} 78
ebpf_solo_io_events_hash{daddr="10.43.165.159",saddr="10.5.0.154"} 2
ebpf_solo_io_events_hash{daddr="127.0.0.1",saddr="127.0.0.1"} 145
ebpf_solo_io_events_hash{daddr="216.239.34.174",saddr="10.5.0.154"} 1
ebpf_solo_io_events_hash{daddr="216.239.36.174",saddr="10.5.0.154"} 1
ebpf_solo_io_events_hash{daddr="74.125.133.100",saddr="10.5.0.154"} 1
ebpf_solo_io_events_hash{daddr="76.76.21.21",saddr="10.5.0.154"} 1
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 4.6054e-05
go_gc_duration_seconds{quantile="0.25"} 8.4773e-05
go_gc_duration_seconds{quantile="0.5"} 0.000114445
go_gc_duration_seconds{quantile="0.75"} 0.0001483
go_gc_duration_seconds{quantile="1"} 0.000223545
go_gc_duration_seconds_sum 0.001830346
go_gc_duration_seconds_count 15
...
```

Como você pode ver, o BumbleBee também fornece métricas no formato Prometheus!

Nos próximos laboratórios, vamos executar este programa eBPF no Kubernetes como um DaemonSet para capturar o tráfego de rede em todos os nós.

Em seguida, vamos raspar as métricas no Prometheus.

E, finalmente, vamos executar um pequeno aplicativo da Web que consultará o Prometheus e o servidor da API do Kubernetes para criar um gráfico de rede.

### `Deploy a app de demonstração Bookinfo`

Precisamos de um aplicativo de demonstração composto por vários microsserviços para demonstrar como o eBPF pode ser usado para exibir as comunicações entre esses serviços. Vamos usar o aplicativo bookinfo para esse fim.

O diagrama abaixo mostra como os diferentes micros serviços se comunicam.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673048510265/342856fa-9085-49b7-a7db-3d14ed961a48.png align="center")

### `Deploy o app bookinfo`

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.11/samples/bookinfo/platform/kube/bookinfo.yaml
```

Use o seguinte comando para verificar o status dos pods:

```bash
kubectl get pods -o wide
```

Você pode ver que temos um cluster Kubernetes de vários nós e que os pods não estão todos em execução no mesmo nó.

### `Deploy Prometheus`

Também queremos manter as métricas que vamos coletar, então vamos realizar o deploy do Prometheus com HELM para essa finalidade.

Vamos usá-lo para armazenar as métricas geradas pelo nosso programa `tcpconnect` eBPF.

Crie o namespace do prometheus:

```bash
kubectl create ns prometheus
```

Instale o Prometheus com HELM

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prom prometheus-community/kube-prometheus-stack --version 30.0.1 -n prometheus
```

### `Deploy BumbleBee`

Criamos uma imagem Docker com BumbleBee. Vamos executá-lo em cada nó do Kubernetes para carregar nosso programa tcpconnect eBPF e coletar informações sobre todas as comunicações que ocorrem no cluster.

Portanto, precisamos implantá-lo como um DaemonSet:

```bash
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: bumblebee
spec:
  selector:
    matchLabels:
      app: bumblebee
  template:
    metadata:
      labels:
        app: bumblebee
    spec:
      containers:
      - name: bumblebee
        image: djannot/bumblebee:0.1
        imagePullPolicy: Always
        command: ["bee"]
        args: ["run", "--insecure", "--plain-http", "ttl.sh/ebpf-solo-tcpconnect:1h"]
        tty: true
        ports:
        - name: http-monitoring
          containerPort: 9091
        securityContext:
          privileged: true
EOF
```

O programa eBPF que enviamos para o Registry do Docker, onde seria fornecido como um argumento: `ttl.sh/ebpf-solo-tcpconnect:1h`

Em seguida, precisamos criar um `PodMonitor` para instruir o **Prometheus** a coletar as métricas do BumbleBee:

```bash
cat << EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: bumblebee
  labels:
    monitoring: bumblebee
    release: prom
spec:
  selector:
    matchLabels:
      app: bumblebee
  jobLabel: bumblebee-stats
  podMetricsEndpoints:
  - port: http-monitoring
    path: /metrics
    interval: 15s
    metricRelabelings:
    - action: replace
      sourceLabels:
      - saddr
      targetLabel: pod_ip
    - action: replace
      sourceLabels:
      - daddr
      targetLabel: cluster_ip
EOF
```

Gere algum tráfego:

```bash
for i in {1..20}; do
    kubectl exec $(kubectl get pods -l app=productpage -o jsonpath='{.items[0].metadata.name}') -- python -c "import requests; r = requests.get('http://localhost:9080/productpage'); print(r.status_code)"
done
```

### `Deploy do kebpf`

Então, agora temos todas as métricas capturadas pelo nosso programa eBPF e armazenadas no **Prometheus**, mas precisamos associar os diferentes endereços IP aos serviços e pods do Kubernetes.

Os endereços IP de origem correspondem aos endereços IP do pod, enquanto o endereço IP de destino corresponde aos endereços IP do cluster de serviço.

Criamos um aplicativo da Web chamado **kebpf** para coletar as métricas do Prometheus e consultar o servidor da API do Kubernetes para vincular tudo.

O aplicativo da web está exibindo o resultado usando uma representação gráfica.

Precisamos criar um `ClusterRoleBinding` para permitir que o container kebpf tenha privilégios administrativos ao cluster:

```bash
cat << EOF | kubectl apply -f -
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kebpf
subjects:
- kind: ServiceAccount
  name: kebpf
  namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```

Agora podemos realizar o deploy do kebpf:

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kebpf
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kebpf
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kebpf
  template:
    metadata:
      labels:
        app: kebpf
    spec:
      serviceAccountName: kebpf
      containers:
      - name: kebpf
        imagePullPolicy: Always
        image: djannot/kebpf:0.1
        command: ["node"]
        args: ["kebpf.js"]
        env:
        - name: PROMETHEUS_ENDPOINT
          value: "http://prom-kube-prometheus-stack-prometheus.prometheus.svc.cluster.local:9090"
        ports:
        - name: http
          containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: kebpf
spec:
  type: LoadBalancer
  selector:
    app: kebpf
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
EOF
```

O diagrama abaixo mostra como todos os componentes funcionam juntos:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673049072371/f212e38d-4017-4e83-a63b-0a7476a7d061.png align="center")

Agora você pode ir para a guia kebpf para ver o gráfico.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673049311613/bafd62d0-efe6-4359-b079-b2b615a81aa4.png align="center")

Como você pode ver, podemos determinar quais serviços se comunicam entre si !

### `BumbleBee do zero`

Agora, vamos explorar o caso de uso do **sistema de arquivos**.

Mas desta vez, vamos começar do zero!

Vamos criar uma versão simplificada da ferramenta **exitsnoop**.

Crie o arquivo chamado `exitsnoop.c` com o seguinte conteúdo:

```c
// SPDX-License-Identifier: (LGPL-2.1 OR BSD-2-Clause) */
/* Copyright (c) 2021 Hengqi Chen */
/* Adapted for `bee` from: https://github.com/iovisor/bcc/blob/master/libbpf-tools/exitsnoop.bpf.c */
#include <vmlinux.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_core_read.h>

#define TASK_COMM_LEN   16

struct event {
        __u32 pid;
        __u32 tid;
        __u32 ppid;
        char comm[TASK_COMM_LEN];
};

struct {
        __uint(type, BPF_MAP_TYPE_RINGBUF);
        __uint(max_entries, 1 << 24);
        __type(value, struct event);
} exits SEC(".maps.print");

SEC("tracepoint/sched/sched_process_exit")
int sched_process_exit(void *ctx)
{
        __u64 pid_tgid = bpf_get_current_pid_tgid();
        __u32 pid = pid_tgid >> 32;
        __u32 tid = (__u32)pid_tgid;

        struct task_struct *task;
        struct event event = {};

        task = (struct task_struct *)bpf_get_current_task();

        event.pid = pid;
        event.tid = tid;
        event.ppid = BPF_CORE_READ(task, real_parent, tgid);
        bpf_get_current_comm(event.comm, sizeof(event.comm));

        struct event *ring_val;

        ring_val = bpf_ringbuf_reserve(&exits, sizeof(struct event), 0);
        if (!ring_val) {
                return 0;
        }

        memcpy(ring_val, &event, sizeof(struct event));

        /* submit event to ringbuf for printing */
        bpf_ringbuf_submit(ring_val, 0);

        return 0;
}

char LICENSE[] SEC("license") = "Dual BSD/GPL";
```

Agora, vamos construir uma imagem a partir do nosso código:

```bash
bee build exitsnoop.c ttl.sh/exitsnoop:1h
```

Em seguida, envie a imagem no Registry local do Docker:

```bash
bee push ttl.sh/exitsnoop:1h
```

Em seguida, execute-o:

```bash
bee run ttl.sh/exitsnoop:1h
```

Você verá todos os processos executando ao lado de seu pid.

Você pode ir para o outro terminal, executar comandos e ver se eles também aparecem.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673130394503/8a4791fb-994d-48da-8e9f-3e20557bc8c8.png align="center")