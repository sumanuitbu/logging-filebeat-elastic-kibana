https://docs.fluentbit.io/manual/installation/kubernetes

https://github.com/fluent/fluent-bit-kubernetes-logging

helm repo add fluent https://fluent.github.io/helm-charts

helm pull fluent/fluent-bit

helm install fluent-bit chart/fluent-bit-0.16.2.tgz --values fluent-bit-values.yaml --namespace logging


