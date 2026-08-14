### Using CLI
1. Create and App register
From GUI or CLI , Use CLI to get Credentials immediatly in JSON Format
```
az ad sp create-for-rbac \
  --name "github-actions-sp" \
  --role "Contributor" \
  --scopes /subscriptions/<YOUR_SUBSCRIPTION_ID> \
  --sdk-auth
```
 
2 Assign permission on aks
```
# Scope at the Subscription Level (Easiest for CI/CD)
az role assignment create \
  --assignee <APP_REGISTRATION_CLIENT_ID> \
  --role "Contributor" \
  --scope /subscriptions/<SUBSCRIPTION_ID>
```
