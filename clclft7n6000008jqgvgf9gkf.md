# eBPF rodando Golang com DaemonSet

O eBPF abre possibilidades de desenvolver ferramentas de observabilidade rodando no Kubernetes.

Pode-se começar com [BCC libbpf-tools](https://github.com/iovisor/bcc/tree/master/libbpf-tools) desenvolvido em C, por exemplo, iniciar o programa `tcpconnlat` e processar seu `stdout` com outro programa para detectar casos em que demorou muito para estabelecer uma conexão TCP.

Na prática, o comando `curl https://andrebassi.com.br` levou 47,80 milissegundos para estabelecer uma conexão, o qual seu IP de origem seria `10.0.2.15` e o IP de destino seria `76.76.21.21`.

```bash
PID    COMM         IP SADDR            DADDR            DPORT LAT(ms)
21168  curl         4  10.0.2.15        76.76.21.21      443   47.80
```

### `Dockerfile`

```bash
FROM ubuntu:groovy as build
RUN apt-get update && \
    apt-get install -y git flex bison llvm cmake clang libclang-dev libelf-dev libcap-dev python3-setuptools && \
    apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
WORKDIR /opt/
RUN git clone https://github.com/iovisor/bcc.git && \
    mkdir ./bcc/build/ && \
    cd ./bcc/build/ && \
    cmake -DPYTHON_CMD=python3 .. && \
    make && \
    cd ../libbpf-tools && \
    make

FROM ubuntu:groovy
RUN apt-get update && \
    apt-get install -y libelf-dev && \
    apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
WORKDIR /opt/
COPY --from=build \
    /opt/bcc/libbpf-tools/ext4dist \
    /opt/bcc/libbpf-tools/runqlen \
    /opt/bcc/libbpf-tools/cpudist \
    /opt/bcc/libbpf-tools/softirqs \
    /opt/bcc/libbpf-tools/filelife \
    /opt/bcc/libbpf-tools/readahead \
    /opt/bcc/libbpf-tools/funclatency \
    /opt/bcc/libbpf-tools/biolatency \
    /opt/bcc/libbpf-tools/biosnoop \
    /opt/bcc/libbpf-tools/llcstat \
    /opt/bcc/libbpf-tools/biopattern \
    /opt/bcc/libbpf-tools/runqslower \
    /opt/bcc/libbpf-tools/xfsslower \
    /opt/bcc/libbpf-tools/numamove \
    /opt/bcc/libbpf-tools/hardirqs \
    /opt/bcc/libbpf-tools/bitesize \
    /opt/bcc/libbpf-tools/opensnoop \
    /opt/bcc/libbpf-tools/runqlat \
    /opt/bcc/libbpf-tools/tcpconnect \
    /opt/bcc/libbpf-tools/cpufreq \
    /opt/bcc/libbpf-tools/drsnoop \
    /opt/bcc/libbpf-tools/vfsstat \
    /opt/bcc/libbpf-tools/biostacks \
    /opt/bcc/libbpf-tools/cachestat \
    /opt/bcc/libbpf-tools/tcpconnlat \
    /opt/bcc/libbpf-tools/syscount \
    /opt/bcc/libbpf-tools/execsnoop \
    ./
```

Crie a imagem docker:

```bash
docker build -t ttl.sh/ebpf-golang:2h .
```

E seu devido push:

```bash
docker push ttl.sh/ebpf-golang:2h
```

### `DaemonSet`

Um `Pod` com recurso de observabilidade deve ser executada em cada node, e assim através de `DaemonSet` garante que todos executem o `tcpconnlat`.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: tcpconnlat-daemon
spec:
  selector:
    matchLabels:
      app: tcpconnlat
  template:
    metadata:
      labels:
        app: tcpconnlat
    spec:
      containers:
        - name: libbpf-tools
          image: ttl.sh/ebpf-golang:2h
          command:
            - /opt/libbpf-tools/tcpconnlat
```

Infelizmente, os pods deram o status como Error, pois os containers não possuem modo privilegiado.

```bash
[1:31:15] andrebassi:~ $ kubectl get pods
NAME                      READY   STATUS   RESTARTS      AGE
tcpconnlat-daemon-np6w8   0/1     Error    3 (36s ago)   64s

[1:32:17] andrebassi:~ $ kubectl logs -f tcpconnlat-daemon-np6w8
2023/01/04 04:32:11 failed to set temporary RLIMIT_MEMLOCK: operation not permitted
```

Por padrão, um container não tem permissão para acessar nenhum dispositivo no host, mas um container "privilegiado" tem acesso a todos os dispositivos no host.

Isso permite que o container tenha todo o mesmo acesso que os processos em execução no host.

[https://kubernetes.io/docs/concepts/security/pod-security-policy/#privileged](https://kubernetes.io/docs/concepts/security/pod-security-policy/#privileged)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: tcpconnlat-daemon
spec:
  selector:
    matchLabels:
      app: tcpconnlat
  template:
    metadata:
      labels:
        app: tcpconnlat
    spec:
      containers:
        - name: libbpf-tools
          image: ttl.sh/ebpf-golang:2h
          command:
            - /opt/libbpf-tools/tcpconnlat
          securityContext:
            privileged: true
```

Ao aplicar o manifesto acima com `securityContext` com `privileged` ativo com `true`, teremos sucesso ao rodar o que precisamos para esse contexto.

```bash
[1:36:29] andrebassi:~ $ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
tcpconnlat-daemon-9sgc5   1/1     Running   0          18s
tcpconnlat-daemon-lrvh5   1/1     Running   0          18s
tcpconnlat-daemon-th9k4   1/1     Running   0          18s

[1:36:18] andrebassi:~ $ kubectl logs -f tcpconnlat-daemon-9sgc5
PID    COMM         IP SADDR            DADDR            DPORT LAT(ms)
5955   coredns      4  127.0.0.1        127.0.0.1        8080  0.02
703    kubelet      4  172.18.8.101     172.18.8.101     6443  0.04
703    kubelet      4  10.233.64.1      10.233.64.4      8181  0.07
703    kubelet      4  169.254.25.10    169.254.25.10    9254  0.03
703    kubelet      4  10.233.64.1      10.233.64.5      8080  0.03
5537   node-cache   4  169.254.25.10    169.254.25.10    9254  0.06
4204   etcd         4  172.18.8.101     172.18.8.103     2380  0.22
4204   etcd         4  172.18.8.101     172.18.8.102     2380  0.21
```

Isso foi um bom exemplo para imaginarmos o que podemos fazer com o poder de observabilidade que o [eBPF](https://ebpf.io/) nos proporciona.