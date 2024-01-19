# frozen_string_literal: true

require 'rake'

desc 'Cleanup minikube objects'
task :delete_stack do
  sh 'kubectl delete all --all'
  sh "kubectl get pvc | grep -v NAME | awk '{print $1}' | xargs kubectl delete pvc"
  sh "kubectl get pv | grep -v NAME | awk '{print $1}' | xargs kubectl delete pv"
  sh 'kubectl get configmap | grep -v "NAME\|kube-root-ca.crt" | awk \'{print $1}\' | xargs kubectl delete configmap'
end
