# Segurança | Múltiplos mTLS da maneira correta com Istio

[Istio](https://istio.io/latest/docs/setup/getting-started/) é um dos melhores ou o melhor Service Mesh existente e open source que conhecemos, por padrão internamente ele vem com total suporte acoplado ao mTLS para a comunicação segura entre pods e também externa.


No Istio, o mTLS é ativado por padrão quando o istio é instalado no seu cluster do Kubernetes. Os modos MTLS são de dois tipos no Istio.

  - Permissivo : por padrão, o modo de MTLS no Istio é permissivo. O modo permissivo pode aceitar tráfego TLS simples e de texto e mútuo.
  - Estrito : este modo garante que o tráfego mtls seja habilitado entre os workloads. Todo o tráfego simples seria interrompido quando o modo Strict fosse ativado.


> Na segurança de informação a maneira mais segura de prover controle de acesso a sua API ou Recurso, seria a troca de chaves, o qual denominamos de TLS Mútuo, o tão conhecido `mTLS`, onde assegura que o tráfego seja seguro e confiável em ambas as direções entre um cliente e um servidor.


  - Quando o mTLS é configurado, o acesso é concedido apenas para solicitações com um certificado de cliente correspondente. Quando uma solicitação chega ao aplicativo, o Istio responde com uma solicitação para o certificado de cliente. 


  - Se o cliente falhar ao apresentar o certificado, a solicitação não terá permissão para continuar. Caso contrário, a troca de chave continua.


  - Neste tutorial vamos abortar desde a instalação do Istio 1.12, a versão mais recente quando isso foi produzido, sua simples configuração e geração dos seus respectivos certificados.


## Instalação

Faça o download do Istio 1.12.2

```
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.12.2 TARGET_ARCH=x86_64 sh -
```

Entrando no diretório que contém o istioctl

```
cd istio-1.12.2/bin
```

Aplique o comando abaixo para verificar qualquer inconsistência no cluster

```
./istioctl x precheck
```

O comando abaixo instala componentes core, habilita secret service discovery, habilita suporte a
forward do IP do client e habilita suporte ao egress para algum eventual controle de saída:

```
./istioctl install \
--set profile=default \
--set values.gateways.istio-ingressgateway.externalTrafficPolicy="Local" \
--set values.gateways.istio-egressgateway.enabled=true
```

Verificando a instalação

```
./istioctl verify-install
```

## Adicionando réplicas para os componentes core

```
kubectl patch hpa -n istio-system istio-ingressgateway -p '{"spec":{"minReplicas": 2}}' --type=merge

kubectl patch hpa -n istio-system istio-egressgateway -p '{"spec":{"minReplicas": 2}}' --type=merge

kubectl patch hpa -n istio-system istiod -p '{"spec":{"minReplicas": 2}}' --type=merge
```

## Instalação de componentes extras

 As ferramentas abaixo instaladas são exclusivas para monitoramento de aplicações pelo istio:

```
cd istio-1.12.2

kubectl apply -f samples/addons/kiali.yaml

kubectl apply -f samples/addons/prometheus.yaml

kubectl apply -f samples/addons/grafana.yaml

kubectl apply -f samples/addons/jaeger.yaml

kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

## Criação do CA | (Certification Authority)

```
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:4096 -subj '/O=andrebassi/CN=*.andrebassi.com.br' -keyout ca.key -out ca.crt
```

> Para melhorar a segurança não use `wildcard` em produção.


## Criação do Certificado

```
openssl req -out mtls.csr -newkey rsa:4096 -nodes -keyout mtls.key -subj "/CN=mtls.andrebassi.com.br/O=andrebassi"
```

## Realizamos assinatura

```
openssl x509 -req -days 365 -CA ca.crt -CAkey ca.key -set_serial 1 -in mtls.csr -out mtls.crt
```

## Verificação do certificado

```
openssl verify -verbose -CAfile ca.crt mtls.crt
```

## Armazenando o CA, a key e certificado em uma secret genérica no cluster

```
kubectl create -n istio-system secret generic mtls-credential \
  --from-file=tls.key=mtls.key \
  --from-file=tls.crt=mtls.crt \
  --from-file=ca.crt=ca.crt
```

## Definindo o Gateway no Istio

```
kind: Gateway
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: mtls-gw
  namespace: istio-system
spec:
  servers:
    - hosts:
        - mtls.andrebassi.com.br
      port:
        name: https
        number: 443
        protocol: HTTPS
      tls:
        credentialName: mtls-credential
        mode: MUTUAL
  selector:
    istio: ingressgateway
```

> Note que ao definir o Gateway no Istio ja atribuimos ao tls, A `credentialName` como `mtls-credential` para o uso como modo `MUTUAL` para mTLS, o qual corresponde ao nome da secret com o CA, key e certificado no passo anterior.


## Definindo agora o VirtualService no Istio

```
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: mtls-vs
  namespace: istio-system
spec:
  hosts:
    - mtls.andrebassi.com.br
  gateways:
    - mtls-gw
  http:
    - route:
        - destination:
            host: productpage.istio-system.svc.cluster.local
            port:
              number: 9080
```

A Route do Destination Host foi definido como `productpage.istio-system.svc.cluster.local` a caráter de demonstração o qual foi instalado na seção acima de Componentes Extra.


## Realizando os testes com a key e o certificado

```
curl -k --cert mtls.crt --key mtls.key --head https://mtls.andrebassi.com.br/
```

Como response teremos:

```
HTTP/2 200
content-type: text/html; charset=utf-8
content-length: 1683
server: istio-envoy
date: Tue, 01 Feb 2022 19:50:01 GMT
x-envoy-upstream-service-time: 6
```

## Realizando os testes com a key, certificado e o CA (Certification Authority)

```
curl --cacert ca.crt --cert mtls.crt --key mtls.key --head https://mtls.andrebassi.com.br/
```

Como response teremos:

```
HTTP/2 200
content-type: text/html; charset=utf-8
content-length: 1683
server: istio-envoy
date: Tue, 01 Feb 2022 19:51:01 GMT
x-envoy-upstream-service-time: 6
```

## Teste fazendo request sem os certificados obteremos esse erro:
```
curl -k --head https://mtls.andrebassi.com.br/

curl: (56) OpenSSL SSL_read: error:1409445C:SSL routines:ssl3_read_bytes:tlsv13 alert certificate required, errno 0
```

## Conclusão

Através desse cenário você pode criar uma chave privada e pública assinada por uma CA representar essas chaves ao servidor para ser verificada, se tudo verificar se a conexão é permitida.

E com único Ingress, no caso o Istio, podemos gerenciar vários CAs e certificados mTLS de diversos parceiros, assim aplicamos mais segurança da maneira correta do que aplicar o uso de IP WhiteList e entre outras prováveis soluções.

Eu gosto de pensar em mTLS como ingressos para um show, pois você tem dois tickets (chave privada e pública) e o servidor tem uma lista (CA) que pode aceitar. 

Depois de verificar matematicamente tudo, você tem permissão para entrar...