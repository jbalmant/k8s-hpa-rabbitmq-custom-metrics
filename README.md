Para termos o HPA funcionando usando custom metrics olhando as filas do RabbiMQ.

# Precisamos de 3 serviços instalados no cluster.

## 1 Prometheus stack
Já temos no cluster.

helm install -f prometheus.values.yaml kube-prometheus-stack prometheus-community/kube-prometheus-stack

## 2 Metrics server
Tem que ver se temos no cluster, mas acho que não temos acesso.

kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

## 3 Prometheus adapter
Não sei se temos no cluster.

helm install -f prometheus-adapter.values.yaml prometheus-adapter prometheus-community/prometheus-adapter

# Após isso, vamos precisar criar alguns recursos:

## RABBITMQ
O rabbitmq já tem builtin os plugins para exportar metricas pro prometheus, é preciso habilitar isso no values do helm.

    metrics:
        enabled: true

    serviceMonitor:
        enabled: true

É possível criar a prometheus rule via values também, mas não é mandatório.
    
Por default o RabbitMQ exporta as metricas das queues aggregadas, para trazer metricas por fila, é precisso habilitar:

    prometheus.return_per_object_metrics = true


**ATENÇÃO** Para o prometheus importar o service monitor, é preciso que a metadata label da rule seja igual as labels que o prometheus procura.


## Prometheus rule
Vamos ter que criar uma Prometheus Rule com os dados da fila que queremos exportar. Ex

    ceil(rabbitmq_queue_messages{queue="my-custom-queue"} / 10)

**ATENÇÃO** Para o prometheus importar essa rule, é preciso que a metadata label da rule seja igual as labels que o prometheus procura.

Valide:
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/services/<service on rule>/<rule>"

## Hpa
É o cara que vai fazer a magia, para funcionar do jeito que estamos pensando, temos que usar a **apiVersion: autoscaling/v2beta2**


**ATENÇÃO**
Tem que estar no mesmo namespace do PrometheusRule
describedObject.apiVersion tem que ser a mesma versão que vem no comando:
    kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/services/<service on rule>/<rule>


## Comandos que podem ajudar:

    kubectl port-forward svc/kube-prometheus-stack-prometheus 8002:9090

    kubectl port-forward --namespace default svc/rabbitmq 9419:9419

    kubectl port-forward svc/rabbitmq 15672:15672


## Links

https://github.com/arielb135/HPA-with-prometheus-and-RabbitMQ
https://www.padok.fr/en/blog/scaling-prometheus-rabbitmq
https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/troubleshooting.md#troubleshooting-servicemonitor-changes
https://www.kloia.com/blog/advanced-hpa-in-kubernetes
https://itnext.io/horizontal-pod-autoscaling-with-custom-metric-from-different-namespace-f19f8446143b