# frozen_string_literal: true

require 'rake'

def minikube_running?
  return true if system('docker ps | grep minikube > /dev/null')

  false
end

desc 'Adjust minikube configs'
task :minikube_config do
  sh 'minikube config set cpus 4'
  sh 'minikube config set memory 8192'
end

desc 'Load docker images into minikube'
task :minikube_load_images, [:file] do |_, args|
  images = File.read(args.file).split("\n")
  images.each do |image|
    sh "minikube image load #{image}" unless image.start_with?('#')
  end
end

desc 'Stop minikube docker instance'
task :minikube_stop do
  sh 'minikube stop' if minikube_running?
end

desc 'Delete minikube docker instance'
task :minikube_delete do
  Rake::Task[:minikube_stop].execute if minikube_running?
  sh 'minikube delete'
end

desc 'generate a pseudo-random key'
task :generate_urandom_key do
  print `tr -dc [:graph:] < /dev/urandom | head -c 32`
end

desc 'Generate Puppet certificate for applications'
task :generate_puppet_cert, [:app_name] do |_, args|
  domain = 'default.svc.cluster.local.pem'
  puppetca = `kubectl get pods | grep puppetca | awk '{print $1}'`.chomp
  puppetdir = '/etc/puppetlabs/puppet/ssl'
  exec_cmd = "kubectl exec -it #{puppetca}"
  secret_cmd = 'kubectl create secret generic'

  puts "Generating puppet certificate for #{args.app_name}.#{domain}"

  gen_cert = <<~HEREDOC
    #{exec_cmd} -- puppetserver ca generate --certname #{args.app_name}.#{domain}
  HEREDOC

  sh gen_cert

  ca_cert = <<~`HEREDOC`
    #{exec_cmd} -- cat #{puppetdir}/ca/ca_crt.pem | base64
  HEREDOC

  key = <<~`HEREDOC`
    #{exec_cmd} -- cat #{puppetdir}/private_keys/#{args.app_name}.#{domain}.pem | base64
  HEREDOC

  cert = <<~`HEREDOC`
    #{exec_cmd} -- cat #{puppetdir}/certs/#{args.app_name}.#{domain}.pem | base64
  HEREDOC

  <<~`HEREDOC`
    #{secret_cmd} #{args.app_name}-ssl-ca-cert --from-literal=#{args.app_name.upcase}_CA_CERT="#{ca_cert}"
  HEREDOC

  <<~`HEREDOC`
    #{secret_cmd} #{args.app_name}-ssl-key --from-literal=#{args.app_name.upcase}_SSL_KEY="#{key}"
  HEREDOC

  <<~`HEREDOC`
    #{secret_cmd} #{args.app_name}-ssl-cert --from-literal=#{args.app_name.upcase}_SSL_CERT="#{cert}"
  HEREDOC
end
