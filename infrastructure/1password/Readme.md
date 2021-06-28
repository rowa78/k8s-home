This uses the helm-chart. I want to monitor all namespaces for annotations. So the serviceaccount in this namespace needs a 'ClusterRoleBinding'. There seems to be a problem with this. See: 
- https://github.com/1Password/connect-helm-charts/issues/54
- https://github.com/1Password/connect-helm-charts/pull/55

Perhaps i find the time to create a pullrequest for that.
