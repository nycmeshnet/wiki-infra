# Wiki Infra

## Backups

```
docker exec 55fb8319dfd0 tar -chvf /config/wiki.tar /app/www/public/uploads/ /app/www/storage/uploads/ /app/www/public/img/
docker exec 00bf1e961d6b mysqldump -u bookstack --password=$THE_REAL_THING bookstackapp > wiki.sql
tar --append -f wiki.tar wiki.sql
gzip < wiki.tar > wiki.tgz
# upload the tgz to s3...
```

## Restore

1. Create `restore.db.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: restorejobdb
  namespace: wiki
spec:
  template:
    spec:
      containers:
      - name: restorejobdb
        image: lscr.io/linuxserver/mariadb
        command:
          - /bin/bash
          - /restore.db.sh
        volumeMounts:
          - name: backup-script
            mountPath: /restore.db.sh
            subPath: restore.db.sh
            readOnly: true
        env:
          - name: RESTORE_S3_URL
            value: "s3://.../backups/wiki.tgz"
          - name: DB_HOST
            valueFrom:
              configMapKeyRef:
                name: wikiconfig
                key: DB_HOST
          - name: DB_USER
            valueFrom:
              secretKeyRef:
                name: wiki-secrets
                key: db-username
          - name: DB_DATABASE
            valueFrom:
              configMapKeyRef:
                name: wikiconfig
                key: DB_DATABASE
          - name: DB_PASS
            valueFrom:
              secretKeyRef:
                name: wiki-secrets
                key: db-password
          - name: AWS_ACCESS_KEY_ID
            valueFrom:
              secretKeyRef:
                name: wiki-secrets
                key: access-key-id
          - name: AWS_SECRET_ACCESS_KEY
            valueFrom:
              secretKeyRef:
                name: wiki-secrets
                key: secret-access-key
      restartPolicy: OnFailure
      volumes:
        - name: backup-script
          configMap:
            name: backup-script
            items:
              - key: restore.db.sh
                path: restore.db.sh
```

2. Create `restore.files.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: restorejobfiles
  namespace: wiki
spec:
  template:
    spec:
      containers:
      - name: restorejobfiles
        image: lscr.io/linuxserver/bookstack:v23.02.2-ls71
        command:
          - /bin/bash
          - /restore.files.sh
        volumeMounts:
          - name: backup-script
            mountPath: /restore.files.sh
            subPath: restore.files.sh
            readOnly: true
          - name: config-vol
            mountPath: /config
          - name: image-uploads-vol
            mountPath: /app/www/public/img
        env:
          - name: RESTORE_S3_URL
            value: "s3://.../backups/wiki.tgz"
          - name: AWS_ACCESS_KEY_ID
            valueFrom:
              secretKeyRef:
                name: wiki-secrets
                key: access-key-id
          - name: AWS_SECRET_ACCESS_KEY
            valueFrom:
              secretKeyRef:
                name: wiki-secrets
                key: secret-access-key
      restartPolicy: OnFailure
      volumes:
        - name: backup-script
          configMap:
            name: backup-script
            items:
              - key: restore.files.sh
                path: restore.files.sh
        - name: config-vol
          persistentVolumeClaim:
            claimName: wikiconfig
        - name: image-uploads-vol
          persistentVolumeClaim:
            claimName: wikiimages
```

2. Run the restore jobs

```
kubectl scale --replicas=0 deployment.apps/wiki-bookstack-helm-bookstack -n wiki
kubectl apply -f restore.files.yaml
# wait for it to complete
kubectl get all -n wiki
kubectl scale --replicas=1 deployment.apps/wiki-bookstack-helm-bookstack -n wiki

kubectl apply -f restore.db.yaml
# wait for it to complete
kubectl get all -n wiki
kubectl scale --replicas=1 deployment.apps/wiki-bookstack-helm-db -n wiki
```

3. If the url is changing, you may need the following:

```
kubectl exec -n wiki -it pod/wiki-bookstack-helm-bookstack-695945475d-mrlhf bash
cd /app/www
php artisan bookstack:update-url https://wiki.mesh.nycmesh.net https://devwiki.mesh.nycmesh.net
```
