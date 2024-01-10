# awx-operator migration

Thanks https://github.com/ansible/awx-operator for the awx-operator.
Thanks https://sezanzeb.github.io/2021/06/16/aws-upgrade-data-restore.html for the database migration step.

This repo purpose is to detail migration steps of AWX from a Docker installation to a K8s installation.

## Migration step:

Get the dump from old docker.
```
cd /some/path/
dockername=awx_postgres
user=awx
database=awx
docker exec $dockername pg_dump -U $user -d $database -f dump.sql && docker cp $dockername:dump.sql .
```

Get the dump from old k8s and transfer it.
```
user=awx
database=awx
kubectl exec -it -n awx awx-demo-postgres-13-0 -- /bin/sh -c "pg_dump -U $user -d $database -f dump.sql" && kubectl cp -n awx awx-demo-postgres-13-0:dump.sql dump.sql
```

Apply kustomization
```
cd /some/path/awx-operator/
kubectl apply -k .
```

Copy the dump to the new postgres container.
```
cd /some/path/awx-operator/
kubectl cp dump.sql -n awx awx-demo-postgres-13-0:/
```

Then exec into the new postgres container to import the dump.
```
# preparation and backup
kubectl exec -it -n awx awx-demo-postgres-13-0 -- /bin/bash
psql -U awx
\connect postgres
# closes all db connections
SELECT pg_terminate_backend (pid) FROM pg_stat_activity WHERE datname = 'awx';
ALTER DATABASE awx RENAME TO awx_empty2;
CREATE DATABASE awx;

# restoring the dump
# ctrl + d exit psql
psql -U awx -d awx < /dump.sql
# takes a few minutes depending of the size of the dump.
```

If needed, execute migration on task pod.
```
k exec -it -n awx $(k get po -n awx | grep task | awk '{print $1}') -- /bin/bash
awx-manage migrate --noinput
```

Following error will happen due to database migration:
```
ERROR    [24b53abeb5364635b0752b16f29101c8] awx.main.utils.encryption Failed to decrypt `Credential(pk=10).ssh_key_data`; if you've recently restored from a database backup or are running in a clustered environment, check that your `SECRET_KEY` value is correct
Traceback (most recent call last):
  File "/var/lib/awx/venv/awx/lib64/python3.9/site-packages/cryptography/fernet.py", line 134, in _verify_signature
    h.verify(data[-32:])
cryptography.exceptions.InvalidSignature: Signature did not match digest.
```

To fix it, need to replace the SECRET_KEY, find the old one:
```
cd /some/path/
cat SECRET_KEY | base64
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==
```

Replace on the secret:
```
k edit secrets -n awx awx-demo-secret-key
```

Restart web and task:
```
k rollout restart deployment -n awx awx-demo-web awx-demo-task
```
