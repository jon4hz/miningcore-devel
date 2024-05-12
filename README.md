# Miningcore Development

This is a guide to run the miningcore fork from 1oopio in k8s for development purposes.

I'll use the following technologies:

- minikube -> to spin up a kubernetes cluster
- argocd -> to deploy miningcore using helm charts
- sealed-secrets -> to encrypt k8s secrets
- kube-prometheus-stack -> to monitor miningcores performance and detect anomalies

> [!Important]
> This guide was not necessarily written with security in mind. So please be more careful with secrets like password and seedphrases in production!

---

- [Installation](./docs/install.md)
- [Building](./docs/build.md)

