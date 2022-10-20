# Kubernetes | Tornando Liveness Probe mais poderoso com Python

[Liveness Probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) usado para controlar a integridade de um aplicativo dentro do container em um pod. 

Quando detecta que algo está errado, ele tem a capacidade de `reciclar` o pod, em outras palavras, terminar aquele pod com algum problema e cria um novo. 

Hoje vamos detalhar uma forma avançada de realizar testes de `Liveness Probe com Python`.


## Por que precisamos disso?

Nosso aplicativo está sendo executado dentro de um container, pois podemos ter uma escala muito maior de réplicas de pods e não podemos garantir manualmente que tudo esteja bem em todos os container...

... Com isso precisamos de alguma forma garantir uma `verificação de integridade`, onde nos garante que o aplicativo esteja íntegro em todos os pods consequentemente.


## O que pode dar errado dentro de um container?

Por algum motivo:

  - Temos problemas de memória;
  - Problemas de uso da CPU;
  - Problemas de sistema operacional;
  - Ou problemas no lado do aplicativo. 
    
Se algum dos motivos seguintes ocorrer, nosso aplicativo não estará disponível e não responderá a nenhuma solicitação. O `Liveness Probe` verifica o estado de integridade do container e, se por algum motivo falhar, o memso será reciclado atomaticamente esse container com o devido problema.


Então podemos configurar em três opções diferentes...


## 1. HTTP Status Code

Opção básica e simples de garantir que nossa API continue funcionando, pois o `Kubelet` enviará uma solicitação `HTTP/GET` para o endpoint do container, e se o código de status de resposta estiver na faixa `(200–400)`, será considerado como um `sucesso` e por ventura qualquer outro código de status será considerado uma `falha`.


## 2. TCP Port

`Kubelet` tentará abrir uma conexão socket via TCP/IP através de alguma porta específica, se for bem-sucedido, será considerado um `sucesso` e caso contrário, será considerado como `falha`.


## 3. Exit Code

O `Kubelet` executará algum comando nos containers, digamos, um `"echo hello-world"`, pois se o comando for bem-sucedido, `retornará 0 código de saída`, que significa que foi um `sucesso` e com isso será considerado como ativo e íntegro, e por fim se o comando retornar um `valor diferente de zero`, significa que algo `falhou` e consequentemente o `Kubelet` reiniciará o pod com o container.

**Essa opção nos permite de desenvolver scripts que verifique tudo o que desejamos e sair com o código de saída adequado.**

Podemos testar alguns procedimentos avançadas dentro do container:

  - Conectividade de CPU;
  - Memória;
  - Disco;
  - API e muito mais.


## Script Python "Tunado"

Desejamos que o script verifique duas abordagens dentro do container:

  - **Average load usage:** Uso médio do uso do CPU do node deve ser `inferior a 90%`, pois qualquer valor acima serão considerados como `falha`;
  - **/health Endpoint:** deve retornar o código de status `200` e qualquer outro código de status será considerado como `falha`;


> O script deve sair com o código de saída `1` quando um dos testes `falhar` e caso contrário, deve sair com `0` informando seu `sucesso`.


Crie um novo arquivo como `liveness_probe.py`
```python
# import required modules
import requests
import sys
import urllib3
import os
import logging

# configure logging
logs_format = '[%(asctime)s] %(levelname)s - %(message)s'
logger = logging.getLogger()

# this is needed in order to see the logs through K8s logs
# '/proc/1/fd/1' file is functioning as the Pod's STDOUT.

handler = logging.FileHandler('/proc/1/fd/1', mode='w')
logger.setLevel(logging.DEBUG)
formatter = logging.Formatter(logs_format)
handler.setFormatter(formatter)
logger.addHandler(handler)

# disable urllib4 warning
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

average_load_value = False
try:
    # fetching the Average load value usage in % 
    average_load_value = os.getloadavg()[0] / os.cpu_count() * 100
    logger.info(f"fetched Average Load value: {average_load_value}")
except Exception as e:
    logger.error(f"Error fetching Average Load Usage value, {e}")

health_api_response = False
try:
    # sending /health request
    health_api_response = requests.get('http://localhost/health', verify=False)
    logger.info(f"got /health response: {health_api_response.status_code}")
except Exception as e:
    logger.error(f"could not get /health response, {e}")

# init tests state
average_load_fail = False
api_test_fail = False

# CPU average load validation
if not average_load_value:
    average_load_fail = True
    logger.error(f"Failed to fetch Average Load value, the test has been failed!")

elif average_load_value >= 90.0:
    average_load_fail = True
    logger.error(f"Average Load value is above threshold, the test has been failed!")
else:
    logger.debug(f"Success Average Load test")

# /health validation
if not health_api_response:
    api_test_fail = True
    logger.error(f"Could not get /health response, the test has been failed!")
elif health_api_response.status_code != 200:
    api_test_fail = True
    logger.error(f"Bad /health response, the test has been failed!")
elif health_api_response.status_code == 200:
    logger.debug(f"Success /health API test,")

# final validation
if api_test_fail:
    logger.error(f"Fail /health API test, Liveness probe has failed!, exiting")
    sys.exit(1)
if average_load_fail:
    logger.error(f"Fail Average load test, Liveness probe has failed!, exiting")
    sys.exit(1)

logger.info(f"all tests passed, Liveness probe has succeed!, exiting")
sys.exit(0)
```

Copie o arquivo python para sua imagem de container no seguinte path
```
/var/liveness_probe.py
```

Agora configuramos um deployment com Liveness probe para verificar o pod a cada 30 segundos.
```
apiVersion: v1
kind: Deployment 
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: apache2-custom
    livenessProbe:
      exec:
        command:
        - python3.7
        - /var/liveness_probe.py
      initialDelaySeconds: 5
      periodSeconds: 30
```

Depois de aplicado, valide a configuração
```
kubectl describe deployment liveness-exec | grep Liveness
```

Teremos

```
Liveness: exec [python3.7 /var/liveness.py]
delay=0s timeout=1s period=30s #success=1 #failure=1
```

Verificando os logs
```
kubectl logs liveness-exec-165f6d5j4-fh9xu | grep ‘Successfully’ -A7
```

Visualizamos

```
[2022-02-02 08:06:53,201] INFO — Successfully fetched Average Load: 3.177083333333333
[2022-02-02 08:06:53,206] INFO — Successfully got /health response: 200
[2022-02-02 08:06:53,207] INFO — Success /health API test,
[2022-02-02 08:06:53,211] INFO — Success Average Load test
[2022-02-02 08:06:53,212] INFO — all tests passed, Liveness probe has succeed!, exiting…
```

**Aparentemente tudo esta rodando com sucesso.**


## Gerando o caos para os testes

Agora vamos quebrar alguma coisa e ver se nosso script consegue interceptar, por exemplo, vamos desligar o servidor web dentro do nosso container.

Como esta aplicação faz uso do webserver do Apache, executaremos dentro do container o comando:
```
sudo service apache2 stop
```

**agora que não temos um web server Apache funcionando, nosso endpoint `/health` não está mais ativo.** 


Vamos verificar nossos logs de container novamente e ver como o script lidou com isso.

```
kubectl logs liveness-exec-165f6d5j4-fh9xu | grep ‘ERROR’ -A7
```

Temos como retorno do log

```
[2021–08–05 08:13:25,207] ERROR — Could not get /health response, the test has been failed!
[2021–08–05 08:13:25,211] ERROR — Fail /health API test, Liveness probe has failed!, exiting”
```

Depois de validarmos que o script lidou com isso, vamos verificar se o Kubernetes está ciente do problema:
```
kubectl describe pod liveness-exec-165f6d5j4-fh9xu | grep ‘Events’ -A5
```

Temos

```
Events:
Type Reason Age From Message
 — — — — — — — — — — — — -
Warning Unhealthy 77s kubelet Liveness probe failed:
Normal Killing 77s kubelet Container liveness-exec failed liveness probe, will be restarted
Normal Created 46s (x2 over 33h) kubelet Created container liveness-exec
Normal Started 46s (x2 over 33h) kubelet Started container liveness-exec
```

O `Liveness Probe` **falhou**, portanto verificaremos se o Kubernetes reinicia o container:
```
kubectl get pods
```

Teremos

```
NAME                           READY   STATUS   RESTARTS  AGE
liveness-exec-875f6c4b4-dh7ag  1/1     Running   1       41s
```

## Conclusões

  - Nesse script, fizemos verificação de CPU `em conjunto` com a checagem do endpoint do healtcheck da API retornando status code `200`;

  - Poderiamos acrecentar a verificação de `TCP Port`, onde poderia constatar se o `Webserver Apache` estaria rodando na `porta 80` por exemplo;

  - Como outra possibilidade, poderia checar se o `banco de dados mySQL está ativo e respondendo`, caso não esteja realizar o `restart` da aplicação para realizar a `re-conexão` com o mesmo;

  - Tudo isso em um único `script Python`, em única abordagem, tornando o `Liveness Probe` bem mais poderodo e tunado!

  - Com a pattern sidecar poderia ser aplicado se o [shareProcessNamespace](https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/) estiver setado como `true`, a partir do `Kubernetes v1.17` e então podemos compartilhar seus processos entre containers dentro do mesmo pod e assim deixando em container isolado esse processo de `Liveness Probe` com script Python, por exemplo:

  ```
  apiVersion: v1
  kind: Pod
  spec:
    containers:
    - name: liveness
      image: apache2-custom
    - name: python-probe
      image: custom-python:3.7
      livenessProbe:
        exec:
          command:
          - python3.7
          - /var/liveness_probe.py
        initialDelaySeconds: 5
        periodSeconds: 30
  ```

**Agora que você sabe o quanto é poderoso esse recurso, tente personalizar o script de acordo com as necessidades do seu aplicativo.**