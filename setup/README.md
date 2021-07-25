Passwort for database (see group_vars/k3s_cluster.yaml) encrypted with

```
echo -n 'letmein' | ansible-vault encrypt_string --stdin-name 'datastore_password'
```