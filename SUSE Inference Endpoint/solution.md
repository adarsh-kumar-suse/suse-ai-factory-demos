# Investigation

## Before - Without editing the blueprint


```
current chart - https://github.com/SUSE/aif/blob/main/charts/aif-operator/files/blueprints/inference-endpoint-litellm-vllm-1.0.0.yaml
```

### Issues - Root cause
-  The mounted volume root is owned by 0:0 with mode 755.
- Container runs as UID 1000, so it cannot write there.
- Because PostgreSQL never initializes, litellm cannot connect to DB and also crash loops.

### other issues -is somebody wants to pay attention
- Insecure Hardcoded Passwords: The fields auth.password and auth.postgres-password contain default strings (ChangeMe-litellm-db-password). Keep these empty if you inject them from external secrets or use automated secret generation.
- not letting the user to select the service type - without that i dont know how the user is goint to access
- no information regarding gpu is being provided as asked
- assuming the user is smart and can create the hugging face and litellm-credentials crediential without document or help texts 



### Fix 
```
 postgresql:
-  architecture: standalone
   auth:
     database: litellm
     username: litellm
     password: ChangeMe-litellm-db-password
-    postgres-password: ChangeMe-litellm-db-password
+    postgresPassword: ChangeMe-litellm-db-password

   persistence:
-    storageClass: local-path
+    storageClassName: local-path

+  podTemplates:
+    initContainers:
+      volume-permissions:
+        enabled: true
```








