## ArgoCD Deployment
Make sure to have below in your values, see [this issue](https://github.com/longhorn/longhorn/issues/4853)
```yaml
preUpgradeChecker:
  jobEnabled: false
```
