# Segurança | Criptografando Secrets no Kubernetes com Sealed Secrets

[Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets/) é uma solução para armazenar dados sensíveis com secrets do [Kubernetes](https://kubernetes.io/) no GIT.


## Comparação com helm-secrets e sops

Alternativa ao Sealed Secrets seria [helm-secrets](https://github.com/zendesk/helm-secrets) que faz o uso do [sops](https://github.com/mozilla/sops) em seu bastidores.

A principal diferença seria:

- Sealed Secrets - descriptografa em *server-side*
- Helm-secrets descriptografa lado do cliente "*client-side*"

A descriptografia do lado do cliente com helm-secrets pode ser um risco de segurança, pois o cliente (como um esteira CI/CD), precisaria ter acesso à chave de criptografia para realizar a implantação.

Com Sealed Secrets, a descriptografia seria do lado do servidor, podemos evitar esse risco de segurança. A chave de criptografia existe apenas no cluster Kubernetes e nunca é exposta.


## Instalação com Helm chart

Sealed Secrets consistem em dois componentes:

- Client-Side CLI para criptografar as secrets
- Server-Side Controller a ser usado para descriptografar Sealed Secrets e criar a secret 

Para instalar usaremos o helm chart oficial de [sealed-secrets repository](https://github.com/bitnami-labs/sealed-secrets/tree/master/helm/sealed-secrets).

Adicionamos o repositório Helm e instalaremos na namespace `kube-system`:

```
helm repo add \
  sealed-secrets https://bitnami-labs.github.io/sealed-secrets \
  && helm repo update
```

```
helm install sealed-secrets \
  --namespace kube-system sealed-secrets/sealed-secrets
```

## CLI tool

Secrets são criptografadas do lado do cliente usando o CLI `kubeseal`.

Para macOS, podemos o brew [fórmula Homebrew](https://formulae.brew.sh/formula/kubeseal). Para Linux, podemos baixar o binário da [página de release do GitHub](https://github.com/bitnami-labs/sealed-secrets/releases).

`macos`
```
brew install kubeseal
```

`linux`
```
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.13.1/kubeseal-linux-amd64 -O kubeseal
```
```
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

`kubeseal` usa o contexto do `kubectl` atual para [acessar o cluster](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/). Antes de continuar, certifique-se de que o `kubectl` esteja conectado ao cluster onde o Sealed Secrets deve ser instalado.


## Criando uma Sealed Secret

`kubeseal` realiza a leitura do manifesto do Kubernetes como entrada de criação de uma `secret`, criptografa e gera um manifesto como saída de uma com o tipo `SealedSecret`.

Neste tutorial, usaremos um manifesto para a criação de uma secret como entrada:

```
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: null
  name: my-secret
data:
  username: Zm9v
  password: YmFy
```

Armazene o manifesto em um arquivo chamado `secret.yaml` e criptografe:

```
cat secret.yaml | kubeseal \
    --controller-namespace kube-system \
    --controller-name sealed-secrets \
    --format yaml \
    > sealed-secret.yaml
```

O conteúdo do arquivo `sealed-secret.yaml` deve ficar assim:

```
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-secret
  namespace: default
spec:
  encryptedData:
    username: AgA...  
    password: AgA...
  template:
    metadata:
      name: my-secret
      namespace: default
```

Devemos agora ter a secret em `secret.yaml` e a secret sealed em `sealed-secret.yaml`.

**Observação**: Não é considerado boa prática armazenar uma secret não criptografada em um arquivo. Isso é apenas para fins de demonstração neste tutorial.

Para implantar o segredo "selado", aplicamos o manifesto com kubectl:

```
kubectl apply -f sealed-secret.yaml
```

Notará que um recurso `SealedSecret` foi criado, descriptografe-o e crie um `Secret` descriptografado.

Vamos verificar se secret esta presente no cluster:

```
kubectl get secret my-secret -o yaml
```

Os dados devem conter nosso username e password codificados em base64:

```
...
data:
  username: Zm9v
  password: YmFy
...
```

## Atualizando a Sealed Secret

Para atualizar um valor em uma secret "selado", teremos que criar um novo manifesto `Secret` localmente e mesclá-lo em um `SealedSecret` existente com a opção `--merge-into`.

No exemplo abaixo, atualizamos o valor da chave de senha (`--from-file=password`) para `1qaz2wsx`.

```
echo -n "1qaz2wsx" \
    | kubectl create secret generic xxx \
        --dry-run=client \
        --from-file=password=/dev/stdin -o json \
    | kubeseal --controller-namespace=kube-system \
        --controller-name=sealed-secrets \
        --format yaml \
        --merge-into sealed-secret.yaml

kubectl apply -f sealed-secret.yaml
```

A secret local é temporária e nome (`xxx` no nosso caso) não importa. O nome da Sealed Secret permanecerá a mesma.

## Adicionando novo valor a Sealed Secret

A diferença entre atualizar um valor e adicionar um novo valor é o nome da chave. Se uma chave chamada `password` já existir, ela será atualizada. Se não existir, ele irá adicioná-lo.

Por exemplo, para adicionar uma nova chave `api_key` (`--from-file=api_key`) em nosso segredo, executamos:

```
echo -n "xsw2zaq1" \
    | kubectl create secret generic xxx \
        --dry-run=client \
        --from-file=api_key=/dev/stdin -o json \
    | kubeseal --controller-namespace=kube-system \
         --controller-name=sealed-secrets \
         --format yaml \
         --merge-into sealed-secret.yaml

kubectl apply -f sealed-secret.yaml
```

## Deletando um valor da Sealed Secret

Para excluir uma Sealed Secret precisamos removê-la do arquivo YAML: 

```
# BSD sed (macOS)
sed -i '' '/api_key:/d' sealed-secret.yaml

# GNU sed
sed -i '/api_key:/d' sealed-secret.yaml

kubectl apply -f sealed-secret.yaml
```

Após aplicar atualizará a `Secret` automaticamente e removerá a `api_key`.


## Excluindo uma Sealed Secret

Para excluir uma secret, usamos kubectl para excluir o recurso:

```
kubectl delete -f sealed-secret.yaml
```

Após aplicar o arquivo, o controlador atualizará o `Secret` e removerá a `api_key`.


## Conclusão

Sealed Secrets é uma maneira segura de gerenciar dados sensíveis no Kubernetes no GI, onde a chave de criptografia é armazenada e as secrets são descriptografados no cluster. O cliente não tem acesso à chave de criptografia.

O cliente usa a ferramenta CLI `kubeseal` para gerar manifestos `SealedSecret` que contêm dados criptografados. Depois de aplicar o arquivo, o controlador do lado do servidor reconhecerá um novo recurso secreto selado e o descriptografará para criar um recurso `Secret`.

No geral, recomendo o uso de "Segredos Selados" para melhorar a segurança.