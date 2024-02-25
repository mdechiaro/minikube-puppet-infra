# Build a Puppet stack in minikube

This started out as weekend project to learn Kubernetes using minikube.
This project creates a multi-node Puppet stack. The configs are managed
with r10k and https://github.com/mdechiaro/puppet-control-repo.

This is not for production use.

## Features

* Puppet stack for config management inside minikube
* Centralized logging with fluentd
* Dashboards to monitor catalogs

## TODO (in no order)
* Fix centralized logging to use something modern
* Add persistent storage
* Add a webhook or self-hosted runner for CI/CD on control repo
* Add helper scripts for cluster management

## Setup minikube image cache to faster deployments

```
minikube image load registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3
minikube image load ghcr.io/voxpupuli/container-puppetdb:8.3.0-latest
minikube image load ghcr.io/voxpupuli/container-puppetserver:8.4.0-latest
minikube image load ghcr.io/voxpupuli/puppetboard:latest
minikube image load fluent/fluentd-kubernetes-daemonset:v1.11.2-debian-syslog-1.0
minikube image load balabit/syslog-ng
minikube image load postgres
```

## Prep

```
minikube config set cpus 4
minikube config set memory 8192
```

## Apply deployment specs in puppet/ directory

```
kubectl apply -f puppet/
```

Use `minikube tunnel` in second terminal to test services with puppet
outside of minikube. Use `kubectl get services` to get those external
IPs. If external IPs are `<pending>`, then the tunnel isn't setup yet.

```
# kubectl get services -o wide
NAME          TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE    SELECTOR
kubernetes    ClusterIP      10.96.0.1       <none>          443/TCP          3h4m   <none>
postgres      ClusterIP      10.108.68.176   <none>          5432/TCP         3h3m   app.kubernetes.io/name=psql
puppet        LoadBalancer   10.99.237.81    10.99.237.81    8140:30886/TCP   3h3m   app.kubernetes.io/name=puppetserver
puppetboard   LoadBalancer   10.105.68.78    10.105.68.78    8080:31206/TCP   52m    app.kubernetes.io/name=puppetboard
puppetca      LoadBalancer   10.102.73.228   10.102.73.228   8140:30449/TCP   3h3m   app.kubernetes.io/name=puppetca
puppetdb      ClusterIP      10.111.26.121   <none>          8081/TCP         3h3m   app.kubernetes.io/name=puppetdb
```

Add `EXTERNAL-IP` for puppet, puppetca services to `/etc/hosts` on the
external agent you want to register to this stack. IPs can change, so
check output of `kubectl get services -A -o wide` to get correct IPs.

```
# /etc/hosts
10.99.237.81 puppet.default.svc.cluster.local
10.102.73.228 puppetca.default.svc.cluster.local
10.105.68.78 puppetboard.default.svc.cluster.local
```

You can verify dns is working with:

```
kubectl exec -i -t dnsutils -- nslookup <service>
```

## Generate Puppet certificates for applications

Some applications require a puppet cert for secure communications with
PuppetDB. The certs need to be generated using `puppetserver ca` cli
commands, and then placed into k8s secrets.

```
kubectl exec -it puppetca-8694f8b54d-5frq9 -- \
  puppetserver ca generate --certname puppetboard.default.svc.cluster.local
```

Add these certs into k8s secrets

```
CA_CERT=$(kubectl exec -it puppetca-8694f8b54d-5frq9 -- cat /etc/puppetlabs/puppet/ssl/ca/ca_crt.pem | base64)
kubectl create secret generic puppetboard-ssl-ca-cert --from-literal=PUPPETBOARD_CA_CERT="${CA_CERT}"
```

```
KEY=$(kubectl exec -it puppetca-8694f8b54d-5frq9 -- cat /etc/puppetlabs/puppet/ssl/private_keys/puppetboard.default.svc.cluster.local.pem | base64)
kubectl create secret generic puppetboard-ssl-key --from-literal=PUPPETBOARD_SSL_KEY="${KEY}"
```

```
CERT=$(kubectl exec -it puppetca-8694f8b54d-5frq9 -- cat /etc/puppetlabs/puppet/ssl/certs/puppetboard.default.svc.cluster.local.pem | base64)
kubectl create secret generic puppetboard-ssl-cert --from-literal=PUPPETBOARD_SSL_CERT="${CERT}"
```

## Cleanup

```
rake delete_stack
```

## Workaround for iptables

If you have docker and lxd / lxc installed on same machine.

```
# https://discuss.linuxcontainers.org/t/lxd-and-docker-firewall-redux-how-to-deal-with-forward-policy-set-to-drop/9953
sudo iptables -I DOCKER-USER  -j ACCEPT
sudo iptables-save
```
