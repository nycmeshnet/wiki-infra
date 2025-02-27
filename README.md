# Wiki Infra

This repository holds the helm chart and github actions workflows used to deploy the [NYC Mesh Wiki](https://wiki.nycmesh.net).

## Updating the Wiki

1. Find the most recent release from [linuxserver/docker-bookstack/releases](https://github.com/linuxserver/docker-bookstack/releases) such as `v25.02-ls195`
2. Update the `tag:` in [values.yaml](./bookstack-helm/values.yaml#L54)
3. Open a pull request with your changes

## Backups

Daily backups are sent to S3.

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
