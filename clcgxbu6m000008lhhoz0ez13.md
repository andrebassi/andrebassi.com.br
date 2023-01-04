# Diferenças entre DevOps e SREs

DevOps é um conjunto de práticas em que o desenvolvimento, as operações de TI, a qualidade e a segurança estão associados à Integração Contínua/Entrega Contínua (CI/CD) para entregar um produto confiável aos clientes finais.

A cultura DevOps facilita a colaboração da equipe principal de desenvolvimento e operação, permitindo que as empresas reduzam os custos corporativos, e lidem com falhas como parte do processo, implementem mudanças graduais, aproveitem ao máximo ferramentas de automação e, por fim, realizam medição de tudo.

**E por isso que você já deve ter ouvido que DevOps não é cargo e sim uma cultura.**

**<mark>DevOps é a união de pessoas, processos e produtos para automatizar a entrega de softwares, viabilizando continuamente a entrega de valor para os usuários.</mark>**

**Diferenças entre DevOps e SREs**  
Enquanto o DevOps trata de qual aspecto das questões, o SRE fala sobre como parte de tudo. No entanto, existem algumas outras diferenças entre os dois.

**1 - Implementação de novos recursos** – DevOps é responsável por implementar a solicitação de novos recursos para um produto, enquanto os SREs garantem que essas novas alterações não aumentem as taxas gerais de falha na produção.

**2 - Fluxo do process**o – Uma equipe de DevOps tem uma perspectiva do ambiente de desenvolvimento para colocar as mudanças do desenvolvimento na produção. Por outro lado, os SREs têm uma perspectiva de produção, para que possam fazer sugestões à equipe de desenvolvimento para limitar as taxas de falha, apesar das novas mudanças.

**3 - Foco** – O foco principal do DevOps é a continuidade e a velocidade do desenvolvimento do produto, enquanto o foco principal do SRE é a confiabilidade, escalabilidade e disponibilidade do sistema.

**4 - Estrutura da Equipe** – Uma equipe típica de DevOps consiste em profissionais com funções e responsabilidades dedicadas, como – Proprietário do Produto, Líder de Equipe, Arquiteto de Nuvem, Desenvolvedor de Software, Engenheiro de QA, Administrador do Sistema. Em contraste, os SREs possuem uma equipe de engenheiros com habilidades operacionais e de desenvolvimento definidas.

#### **Habilidades principais do DevOps**

#### Embora não seja uma regra que para ser um DevOps você seja um administrador de sistemas ou desenvolvedor, é muito importante que você tenha experiência em ambos os lados.

* CI/CD (Continuous Integration CI e Continuous Delivery CD);
    
* Infraestrutura como código (IaC);
    
* Containers e orquestração;
    
* Gerenciamento de serviços em nuvem;
    
* Scripts;
    
* Automação e testes;
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1672791427599/0ed5844e-d7de-412d-8533-c3a880861e04.png align="right")

**Diferença nas funções de trabalho de SRE e DevOps**

<table><tbody><tr><td colspan="1" rowspan="1"><p><strong>DevOps</strong></p></td><td colspan="1" rowspan="1"><p><strong>Site Reliability Engineer (SRE)</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p>Entrega</p></td><td colspan="1" rowspan="1"><p>Operações</p></td></tr><tr><td colspan="1" rowspan="1"><p>Automação</p></td><td colspan="1" rowspan="1"><p>Resposta a incidentes</p></td></tr><tr><td colspan="1" rowspan="1"><p>Criação de ambientes</p></td><td colspan="1" rowspan="1"><p>Post Mortems</p></td></tr><tr><td colspan="1" rowspan="1"><p>Gerenciamento de configurações</p></td><td colspan="1" rowspan="1"><p>Monitorar eventos e alertas</p></td></tr><tr><td colspan="1" rowspan="1"><p>Infraestrutura como código</p></td><td colspan="1" rowspan="1"><p>Plano de capacidade</p></td></tr><tr><td colspan="1" rowspan="1"><p>Foco primário: Velocidade de entrega</p></td><td colspan="1" rowspan="1"><p>Foco primário: Confiabildiade</p></td></tr></tbody></table>

### `Problemas resolvidos pelas equipes de DevOps`

A implementação das práticas de DevOps pode reduzir o atrito entre as equipes de desenvolvimento e operações. Ele também pode ajudá-lo a entregar o produto final de forma confiável junto com outros desafios e problemas que as equipes de DevOps podem resolver.

**1 - Custo reduzido de desenvolvimento e manutenção**  
Uma equipe de DevOps sempre trabalha para CI/CD, colocando mais esforço em testes automatizados em vez de testes manuais e melhorando o gerenciamento de lançamentos automatizando tudo.

Quanto ao Ciclo de Vida de Desenvolvimento de Software tradicional, sempre há uma labuta no esforço (desenvolvimento, teste, liberação), o que aumenta o custo geral para o desenvolvimento do produto e manutenção da produção. Colocar a prática do DevOps em execução pode reduzir significativamente o tempo de entrega, desenvolvimento e custos de manutenção.

**2 - Ciclo de lançamento mais curto**  
Uma das mudanças mais eficazes que uma equipe de DevOps traz é entregar mais rápido com um ciclo de lançamento mais curto. A razão pela qual a equipe de DevOps defende um ciclo de lançamento mais curto é que é fácil de gerenciar e reverter para a versão estável caso haja algum problema.

Ao contrário dos ciclos de lançamento tradicionais, onde o foco é entregar tudo em um único lançamento, o que aumenta o risco de falha na produção é muito mais difícil reverter. Se as práticas de DevOps forem seguidas rigorosamente, a organização sempre terá um sistema de versão de lançamento adequado com versões de lançamento e intervenções manuais mínimas com os artefatos de lançamento.

Aqui estão os ganhos de um ciclo de lançamento mais curto:

* Entregue a nova solicitação de mudança com mais frequência;
    
* Enviar as atualizações (correções de bugs, patches de segurança, atualizações de versão) para a produção é muito mais fácil.
    

**3 - Automated and Continuous Testing**  
Ao contrário do ciclo de desenvolvimento tradicional, em que a equipe de teste precisa aguardar a entrega do produto no ambiente de teste para iniciar o DevOps de teste, o teste é injetado desde o início do ciclo de vida do desenvolvimento.

O DevOps facilita testes contínuos e automatizados com a ajuda da ferramenta CI/CD (Jenkins) e controle de versão (Git, BitBucket). A cobertura adequada de testes funcionais, não funcionais e de interação em execução nos pipelines pode melhorar significativamente os aspectos de automação de teste do projeto.

  

**4 -  Automação em tudo**  
A automação é um dos maiores desafios que a equipe de SRE enfrenta. Muitas vezes é observado que rollouts e tarefas de suporte são realizadas manualmente, levando a inconsistência e aumentando a probabilidade de erro humano.

Uma boa prática para gerenciar a infraestrutura é usar Infrastructure as Code (IaC) com a ajuda de Terraform, Pulumi e as ferramentas de automação como Ansible, Puppet, Chef. A equipe SRE pode aproveitar essas ferramentas para resolver o problema de automação.

Em geral, uma equipe SRE é responsável pela disponibilidade, latência, desempenho, eficiência, gerenciamento de mudanças, monitoramento, resposta a emergências e planejamento de capacidade dos serviços sob supervisão.

### `Problemas que as equipes de SREs resolvem`

![www](https://cdn.hashnode.com/res/hashnode/image/upload/v1672791514524/5f5b3ddd-cd1f-48f1-8d44-022a07e3f257.png align="left")

**1 -  Tempo médio de recuperação reduzido (MTTR)**  
A equipe SRE é responsável por manter a produção funcionando. No caso de um bug ou falha de produção, as equipes SRE podem reverter para a versão estável anterior de um produto para que o tempo médio de recuperação (MTTR) seja reduzido.

**2 - Tempo médio de detecção reduzido (MTTD)**  
O outro problema que a equipe do SRE está tentando resolver é reduzir o tempo médio de detecção (MTTD) usando os lançamentos Canary para que o novo lançamento seja disponibilizado para um pequeno grupo de usuários antes de fazer lançamentos completos. Os lançamentos Canary ajudam a equipe SRE a encontrar os problemas no estágio inicial com um número limitado de usuários afetados.

**3\. Documentação de Chamadas e Incidentes**  
Freqüentemente, os engenheiros de confiabilidade (SRE) precisam assumir as funções de plantão para gerenciar incidentes imprevistos, mas também precisam preparar a documentação dos incidentes e as etapas de solução de problemas para que possam ajudar outras pessoas a realizar as tarefas de plantão.

A equipe SRE pode construir uma valiosa base de conhecimento sobre incidentes para melhorar o tempo de solução de problemas do incidente.

  
**4\. Conhecimento Compartilhado**  
Ganhar exposição e construir a base de conhecimento do ecossistema de desenvolvimento de produtos (desenvolvimento, teste, estágio, prod) é sempre benéfico para os engenheiros de confiabilidade preverem os problemas no ambiente de produção.

Mas o principal problema surge quando a base de conhecimento está desatualizada, os playbooks de automação têm comentários irrelevantes. Atualizações regulares da base de conhecimento por SREs em colaboração com DevOps podem preencher a lacuna de conhecimento entre as equipes.

#### De acordo com o Google, os princípios fundamentais do SRE são:

* **Adoção do risco:** fornece abordagens neutras ao Service Management usando orçamentos de erro;
    
* **Objetivos de nível de serviço:** fornece recomendações para indicadores desintegrados a partir de contratos e examina como o SRE usa os termos.
    
* **Eliminação de esforço:** afastando-se de tarefas mundanas e repetitivas que são desprovidas de valor.
    
* **Monitoramento de sistemas distribuídos:** sempre evite fechar os olhos para o que está acontecendo na organização por causa da confiabilidade.
    
* **Engenharia de versões:** considere cuidadosamente as versões para garantir que elas sejam consistentes e não contribuam com indisponibilidades.
    
* **Simplicidade:** um sistema muito complexo pode reduzir a confiabilidade e dificultar o dimensionamento para um local mais simples.
    

### `Conclusão`

A verdade é que SRE e DevOps têm muitas coisas em comum, ambos são metodologias implementadas para monitorar e garantir que tudo funcione conforme o esperado.

Embora os dois compartilham alguns valores essenciais, o foco de seu trabalho é diferente, pois o ciclo de vida do aplicativo por meio de DevOps e o gerenciamento do ciclo de vida de operações por meio de SRE.

No entanto, ambos conectam as equipes de Desenvolvimento e Operação, compartilhando responsabilidades semelhantes. E ambos estão trabalhando com o mesmo objetivo, aprimorar o ciclo de lançamento e alcançar maior confiabilidade do produto.