---
layout: post
title: Kubernetes Dashboard
---

```
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

Now, you can run the `kubectl proxy` and then access the dashboard at:

http://127.0.0.1:8001/ui

However, the web dashboard now requires authentication, and you should review
[this on Stack Overflow](https://stackoverflow.com/questions/46664104/how-to-sign-in-kubernetes-dashboard). 
The gist of it is to do the following:

```
cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
EOF
```

And, then simply skip the authentication option on accessing the URL.
