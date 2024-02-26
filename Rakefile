# frozen_string_literal: true

require 'rake'

desc 'Adjust minikube configs'
task :minikube_config do
  sh 'minikube config set cpus 4'
  sh 'minikube config set memory 8192'
end

desc 'Start minikube docker instance'
task :minikube_start do
  sh 'minikube start'
end

desc 'Load docker images into minikube'
task :minikube_load_images do
  images = [
    'balabit/syslog-ng:latest',
    'fluent/fluentd-kubernetes-daemonset:v1.11.2-debian-syslog-1.0',
    'ghcr.io/voxpupuli/container-puppetdb:8.3.0-main',
    'ghcr.io/voxpupuli/container-puppetserver:8.4.0-main',
    'ghcr.io/voxpupuli/puppetboard:latest',
    'postgres:latest',
    'registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3'
  ]
  images.each do |image|
    sh "minikube image load #{image}"
  end
end

desc 'Stop minikube docker instance'
task :minikube_stop do
  sh 'minikube stop'
end

desc 'Delete minikube docker instance'
task :minikube_delete do
  Rake::Task[:minikube_stop].execute
  sh 'minikube delete'
end

desc 'Apply kubectl configs'
task :kubectl_apply, [:config] do |_, args|
  `echo 'Applying configs in #{args.config}'`
  sh "kubectl apply -f #{args.config}"
end

desc 'Generate Puppet certificate for applications'
task :generate_puppet_cert, [:app_name] do |_, args|
  domain = 'default.svc.cluster.local.pem'
  puppetca = `kubectl get pods | grep puppetca | awk '{print $1}'`.chomp

  `echo 'Generating puppet certificate for #{args.app_name}.#{domain}'`

  sh "kubectl exec -it #{puppetca} -- puppetserver ca generate --certname #{args.app_name}.#{domain}"

  puppetdir = '/etc/puppetlabs/puppet/ssl'
  ca_cert = `kubectl exec -it #{puppetca} -- cat #{puppetdir}/ca/ca_crt.pem | base64`
  key = `kubectl exec -it #{puppetca} -- cat #{puppetdir}/private_keys/#{args.app_name}.#{domain} | base64`
  cert = `kubectl exec -it #{puppetca} -- cat #{puppetdir}/certs/#{args.app_name}.#{domain} | base64`

  kubectl_secret_cmd = 'kubectl create secret generic'
  `#{kubectl_secret_cmd} #{args.app_name}-ssl-ca-cert --from-literal=#{args.app_name.upcase}_CA_CERT='#{ca_cert}'`
  `#{kubectl_secret_cmd} #{args.app_name}-ssl-key --from-literal=#{args.app_name.upcase}_SSL_KEY='#{key}'`
  `#{kubectl_secret_cmd} #{args.app_name}-ssl-cert --from-literal=#{args.app_name.upcase}_SSL_CERT='#{cert}'`
end
