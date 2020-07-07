# Backup a folder from a Kubernetes Pod to an S3 bucket

## Example for backing up GeoServer's data folder

```bash
NAMESPACES=dev-csvs,stage-csvs

OIFS=$IFS
IFS=','
for ns in ${NAMESPACES}
do
    kubectl create role pod-reader --verb=get --verb=list --verb=watch --resource=pods --namespace=${ns}
    kubectl create rolebinding ${ns}-pod-reader --role=pod-reader --serviceaccount=default:default --namespace=${ns}
    kubectl create role pod-exec --verb=create --resource=pods/exec --namespace=${ns}
    kubectl create rolebinding ${ns}-pod-exec --role=pod-exec --serviceaccount=default:default --namespace=${ns}
done

IFS=$OIFS
```

```bash
cat >geoserver-config-backup.yaml <<EOF
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: geoserver-config-backup
spec:
  schedule: "0 0 * * *"
  jobTemplate:
    spec:
      ttlSecondsAfterFinished: 3600
      template:
        spec:
          containers:
          - name: geoserver-config-backup
            image: satapps/backup-pod-folder-to-s3:0.2.0
            args:
            - /backup-folder.sh
            env:
            - name: DEBUG
              value: ""
            - name: NAMESPACES
              value: "dev-csvs,stage-csvs"
            - name: SELECTOR
              value: "app.kubernetes.io/name=geoserver"
            - name: BACKUP_FOLDER
              value: "/geoserver_data/data"
            - name: BACKUP_NAME_TEMPLATE
              value: "geoserver_config_backup"
            - name: AWS_DESTINATION_BUCKET
              value: "csvs-backups"
            - name: AWS_ENDPOINT_URL
              value: "http://s3-uk-1.sa-catapult.co.uk"
            - name: AWS_ACCESS_KEY_ID
              value: "AKIAIOSFODNN7INVALID"
            - name: AWS_SECRET_ACCESS_KEY
              value: "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"
          restartPolicy: OnFailure
EOF

kubectl apply -f geoserver-config-backup.yaml
```
