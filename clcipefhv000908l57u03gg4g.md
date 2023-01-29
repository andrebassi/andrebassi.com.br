# Talos Linux

**Talos Linux** é uma abordagem interessante para um sistema operacional Linux baseado em API.

Em vez de usar SSH para conectar e executar comandos do sistema operacional, o Talos Linux não fornece nenhum acesso SSH.

Toda a configuração é feita por meio de um endpoint [gRPC-API](https://grpc.io/).

* Imutável;
    
* Atômico;
    
* Efêmero;
    
* Mínimo;
    
* Seguro por padrão;
    
* Gerenciado por meio de um único arquivo de configuração declarativa e API gRPC;
    
* Pode ser implantado em plataformas de container, clouds, Máquinas Virtuais e principalmente ambientes On-Premise;
    
* Atualização de versão do Kernel, Container Runtime e o próprio Kubernetes de maneira transparente zero downtime;
    

### `Demonstrações com os Engenheiros`

No vídeo a seguir o Engenheiro responsável demonstra como é esse processo de atualização do Linux:

%[https://www.youtube.com/watch?v=AAF6WhX0USo] 

E nesse vídeo a seguir a atualização do Kubernetes, para atender sua evolução, manutenção e fatores de segurança em geral:

%[https://www.youtube.com/watch?v=uOKveKbD8MQ] 

### `Essa abstração toda tem muitas vantagens, mas também alguns desafios:`

* Sem nenhum acesso SSH e sem maneira de apenas verificar e configurar "algo";
    
* Acesso 100% com Bash CLI, desde sua configuração, atualização, observabilidade e sustentação;
    

### `Mais detalhes sobre?`

A empresa se chama [Sideros Labs](https://www.siderolabs.com/), o qual o Talos Linux é seu produto "Carro Chefe".

Atualmente eles entraram no Sandbox da CNFC ([`Cloud Native Computing Foundation`](https://www.cncf.io/)) como Instalação de Kubernetes Certificada:

[https://landscape.cncf.io/card-mode?category=certified-kubernetes-installer&grouping=category&selected=sidero-talos-linux](https://landscape.cncf.io/card-mode?category=certified-kubernetes-installer&grouping=category&selected=sidero-talos-linux)

### `Mais um vídeo bacana`

Veja o vídeo do canal [DevOps Toolkit](https://www.youtube.com/@DevOpsToolkit) aonde faz um ótimo review e uma demonstração na cloud da [Digital Ocean](https://cloud.digitalocean.com/) sobre o **Talos Linux**:

%[https://www.youtube.com/watch?v=iEFb2Zg4xUg] 

### `Alguns casos de sucesso`

* Kubernetes rodando em ambiente OpenStack da RedHat;
    
* Utilização com ambiente Efêmero dentro da esteira DevOps de CI do Gitlab para realização de testes unitários e regressivos;
    

### `Mais informações`

O Talos Linux [ja está em sua versão 1.3.1](https://github.com/siderolabs/talos/releases) (28-12-2022), já homologado e testado na [versão do Kubernetes 1.26.0](https://github.com/siderolabs/kubelet/releases/tag/v1.26.0) e continua evoluindo constantemente, o qual também já está totalmente compatível com os recursos de observabilidade eBPF.

Para maiores detalhes visite a Matrix de compatibilidade no link abaixo:  
[https://www.talos.dev/v1.3/introduction/support-matrix/](https://www.talos.dev/v1.3/introduction/support-matrix/)