## Gogs Deployment on restricted network environment

### 1. 필요한 이미지 및 skopeo copy 명령어

인터넷이 되는 서버에서 다음 명령어를 실행하여 인터넷이 제한된 private registry로 이미지를 복사합니다.

- 이미지 복사

  ```bash
  skopeo copy --dest-creds admin:r3dh4t1! --dest-tls-verify=false docker://docker.io/openshiftdemos/gogs:latest docker://ext-registry.rhocp4.nirs.go.kr:5000/gogs:latest
  
  skopeo copy --src-creds rhn-support-hyou:You2707you! --dest-creds admin:r3dh4t1! --dest-tls-verify=false docker://registry.redhat.io/rhscl/postgresql-96-rhel7:latest docker://ext-registry.rhocp4.nirs.go.kr:5000/postgresql-96-rhel7:latest
  ```

- 이미지 import

  oc import-image 명령어를 사용하여 OpenShift Cluster 환경에 image straeam으로 이미지를 import 합니다.

  ```bash
  oc import-image postgresql-96-rhel7:latest --from=ext-registry.rhocp4.nirs.go.kr:5000/postgresql-96-rhel7:latest --confirm --insecure -n openshift
  oc import-image gogs:latest --from=ext-registry.rhocp4.nirs.go.kr:5000/gogs:latest --confirm --insecure -n openshift
  oc import-image gogs:latest --from=ext-registry.rhocp4.nirs.go.kr:5000/gogs:latest --confirm --insecure -n gogs
  ```



### 2. Gogs Template Deployment

Gogs를 OpenShift 환경에 쉽게 배포하기 위해 template을 작성하여 적용합니다.

- gogs template 작성 

  ```bash
  cat << EOF > gogs-template.yaml
  kind: Template
  apiVersion: v1
  metadata:
    annotations:
      description: The Gogs git server (https://gogs.io/)
      tags: instant-app,gogs,go,golang
    name: gogs
  objects:
  - kind: ServiceAccount
    apiVersion: v1
    metadata:
      creationTimestamp: null
      labels:
        app: ${APPLICATION_NAME}
      name: ${APPLICATION_NAME}
  - kind: Service
    apiVersion: v1
    metadata:
      annotations:
        description: Exposes the database server
      name: ${APPLICATION_NAME}-postgresql
      labels:
        app: ${APPLICATION_NAME}
    spec:
      ports:
      - name: postgresql
        port: 5432
        targetPort: 5432
      selector:
        name: ${APPLICATION_NAME}-postgresql
  - kind: DeploymentConfig
    apiVersion: v1
    metadata:
      annotations:
        description: Defines how to deploy the database
      name: ${APPLICATION_NAME}-postgresql
      labels:
        app: ${APPLICATION_NAME}
    spec:
      replicas: 1
      selector:
        name: ${APPLICATION_NAME}-postgresql
      strategy:
        type: Recreate
      template:
        metadata:
          labels:
            name: ${APPLICATION_NAME}-postgresql
          name: ${APPLICATION_NAME}-postgresql
        spec:
          serviceAccountName: ${APPLICATION_NAME}
          containers:
          - env:
            - name: POSTGRESQL_USER
              value: ${DATABASE_USER}
            - name: POSTGRESQL_PASSWORD
              value: ${DATABASE_PASSWORD}
            - name: POSTGRESQL_DATABASE
              value: ${DATABASE_NAME}
            - name: POSTGRESQL_MAX_CONNECTIONS
              value: ${DATABASE_MAX_CONNECTIONS}
            - name: POSTGRESQL_SHARED_BUFFERS
              value: ${DATABASE_SHARED_BUFFERS}
            - name: POSTGRESQL_ADMIN_PASSWORD
              value: ${DATABASE_ADMIN_PASSWORD}
            image: ' '
            livenessProbe:
              initialDelaySeconds: 30
              tcpSocket:
                port: 5432
              timeoutSeconds: 1
              failureThreshold: 10
              periodSeconds: 20
            name: postgresql
            ports:
            - containerPort: 5432
            readinessProbe:
              exec:
                command:
                - /bin/sh
                - -i
                - -c
                - psql -h 127.0.0.1 -U ${POSTGRESQL_USER} -q -d ${POSTGRESQL_DATABASE} -c 'SELECT 1'
              initialDelaySeconds: 30
              timeoutSeconds: 1
              failureThreshold: 10
            resources:
              limits:
                memory: 512Mi
            volumeMounts:
            - mountPath: /var/lib/pgsql/data
              name: gogs-postgres-data
          volumes:
          - name: gogs-postgres-data
            persistentVolumeClaim:
              claimName: gogs-postgres-data
      triggers:
      - imageChangeParams:
          automatic: true
          containerNames:
          - postgresql
          from:
            kind: ImageStreamTag
            name: postgresql-96-rhel7:${DATABASE_VERSION}
            namespace: openshift
        type: ImageChange
      - type: ConfigChange
  - kind: Service
    apiVersion: v1
    metadata:
      annotations:
        description: The Gogs server's http port
        service.alpha.openshift.io/dependencies: '[{"name":"${APPLICATION_NAME}-postgresql","namespace":"","kind":"Service"}]'
      labels:
        app: ${APPLICATION_NAME}
      name: ${APPLICATION_NAME}
    spec:
      ports:
      - name: 3000-tcp
        port: 3000
        protocol: TCP
        targetPort: 3000
      selector:
        app: ${APPLICATION_NAME}
        deploymentconfig: ${APPLICATION_NAME}
      sessionAffinity: None
      type: ClusterIP
    status:
      loadBalancer: {}
  - kind: Route
    apiVersion: v1
    id: ${APPLICATION_NAME}-http
    metadata:
      annotations:
        description: Route for application's http service.
      labels:
        app: ${APPLICATION_NAME}
      name: ${APPLICATION_NAME}
    spec:
      host: ${HOSTNAME}
      to:
        name: ${APPLICATION_NAME}
  - kind: DeploymentConfig
    apiVersion: v1
    metadata:
      labels:
        app: ${APPLICATION_NAME}
      name: ${APPLICATION_NAME}
    spec:
      replicas: 1
      selector:
        app: ${APPLICATION_NAME}
        deploymentconfig: ${APPLICATION_NAME}
      strategy:
        resources: {}
        rollingParams:
          intervalSeconds: 1
          maxSurge: 25%
          maxUnavailable: 25%
          timeoutSeconds: 600
          updatePeriodSeconds: 1
        type: Rolling
      template:
        metadata:
          creationTimestamp: null
          labels:
            app: ${APPLICATION_NAME}
            deploymentconfig: ${APPLICATION_NAME}
        spec:
          serviceAccountName: ${APPLICATION_NAME}
          containers:
          - image: " "
            imagePullPolicy: Always
            name: ${APPLICATION_NAME}
            ports:
            - containerPort: 3000
              protocol: TCP
            resources: {}
            terminationMessagePath: /dev/termination-log
            volumeMounts:
            - name: gogs-data
              mountPath: /opt/gogs/data
            - name: gogs-config
              mountPath: /etc/gogs/conf
            readinessProbe:
                httpGet:
                  path: /
                  port: 3000
                  scheme: HTTP
                initialDelaySeconds: 40
                timeoutSeconds: 1
                periodSeconds: 20
                successThreshold: 1
                failureThreshold: 10
            livenessProbe:
                httpGet:
                  path: /
                  port: 3000
                  scheme: HTTP
                initialDelaySeconds: 40
                timeoutSeconds: 1
                periodSeconds: 10
                successThreshold: 1
                failureThreshold: 10
          dnsPolicy: ClusterFirst
          restartPolicy: Always
          securityContext: {}
          terminationGracePeriodSeconds: 30
          volumes:
          - name: gogs-data
            persistentVolumeClaim:
              claimName: gogs-data
          - name: gogs-config
            configMap:
              name: gogs-config
              items:
                - key: app.ini
                  path: app.ini
      test: false
      triggers:
      - type: ConfigChange
      - imageChangeParams:
          automatic: true
          containerNames:
          - ${APPLICATION_NAME}
          from:
            kind: ImageStreamTag
            name: ${APPLICATION_NAME}:${GOGS_VERSION}
        type: ImageChange
  - kind: ImageStream
    apiVersion: v1
    metadata:
      labels:
        app: ${APPLICATION_NAME}
      name: ${APPLICATION_NAME}
    spec:
      tags:
      - name: "${GOGS_VERSION}"
        from:
          kind: DockerImage
          name: ext-registry.rhocp4.nirs.go.kr:5000/gogs:${GOGS_VERSION}
        importPolicy: {}
        annotations:
          description: The Gogs git server docker image
          tags: gogs,go,golang
          version: "${GOGS_VERSION}"
  - kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: gogs-data
      labels:
        app: ${APPLICATION_NAME}
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: ${GOGS_VOLUME_CAPACITY}
  - kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: gogs-postgres-data
      labels:
        app: ${APPLICATION_NAME}
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: ${DB_VOLUME_CAPACITY}
  - kind: ConfigMap
    apiVersion: v1
    metadata:
      name: gogs-config
      labels:
        app: ${APPLICATION_NAME}
    data:
      app.ini: |
        RUN_MODE = prod
        RUN_USER = gogs
  
        [database]
        DB_TYPE  = postgres
        HOST     = ${APPLICATION_NAME}-postgresql:5432
        NAME     = ${DATABASE_NAME}
        USER     = ${DATABASE_USER}
        PASSWD   = ${DATABASE_PASSWORD}
  
        [repository]
        ROOT = /opt/gogs/data/repositories
  
        [server]
        ROOT_URL=http://${HOSTNAME}
        SSH_DOMAIN=${HOSTNAME}
  
        [security]
        INSTALL_LOCK = ${INSTALL_LOCK}
  
        [service]
        ENABLE_CAPTCHA = false
  
        [webhook]
        SKIP_TLS_VERIFY = ${SKIP_TLS_VERIFY}
  parameters:
  - description: The name for the application.
    name: APPLICATION_NAME
    required: true
    value: gogs
  - description: 'Custom hostname for http service route.  Leave blank for default hostname, e.g.: <application-name>-<project>.<default-domain-suffix>'
    name: HOSTNAME
    required: true
  - description: Volume space available for data, e.g. 512Mi, 2Gi
    name: GOGS_VOLUME_CAPACITY
    required: true
    value: 1Gi
  - description: Volume space available for postregs data, e.g. 512Mi, 2Gi
    name: DB_VOLUME_CAPACITY
    required: true
    value: 1Gi
  - displayName: Database Username
    from: gogs
    value: gogs
    name: DATABASE_USER
  - displayName: Database Password
    from: '[a-zA-Z0-9]{8}'
    value: gogs
    name: DATABASE_PASSWORD
  - displayName: Database Name
    name: DATABASE_NAME
    value: gogs
  - displayName: Database Admin Password
    from: '[a-zA-Z0-9]{8}'
    generate: expression
    name: DATABASE_ADMIN_PASSWORD
  - displayName: Maximum Database Connections
    name: DATABASE_MAX_CONNECTIONS
    value: "100"
  - displayName: Shared Buffer Amount
    name: DATABASE_SHARED_BUFFERS
    value: 12MB
  - displayName: Database version (PostgreSQL)
    name: DATABASE_VERSION
    value: "9.6"
  - name: GOGS_VERSION
    displayName: Gogs Version
    description: 'Version of the Gogs container image to be used (check the available version https://hub.docker.com/r/openshiftdemos/gogs/tags)'
    value: "latest"
    required: true
  - name: INSTALL_LOCK
    displayName: Installation lock
    description: 'If set to true, installation (/install) page will be disabled. Set to false if you want to run the installation wizard via web'
    value: "true"
  - name: SKIP_TLS_VERIFY
    displayName: Skip TLS verification on webhooks
    description: Skip TLS verification on webhooks. Enable with caution!
    value: "false"
  EOF
  ```

- template deployment

  ```bash
  oc apply -f gogs-template.yaml
  ```



### 3. Gogs Deployment

- 프로젝트 생성

  gogs를 배포할 프로젝트를 생성합니다. 콘솔 또는 CLI를 사용합니다.

  ```bash
  oc new-project gogs
  ```

- gogs deployment

  콘솔 접속 > 개발자 콘솔 변경 > + Add > 개발자 카탈로그 > 모든 서비스 > gogs 검색 > gogs template를 사용하여 배포 진행

  ![01_gogs_template](https://github.com/justone0127/Gogs-Deployment-on-restricted-network-enviornment/tree/main/images)

- gogs topolozy 

  배포가 완료되면 다음과 같이 `gogs-postgresql` Pod와 `gogs` Pod가 정상적으로 올라옵니다.

  ![02_topolozy](https://github.com/justone0127/Gogs-Deployment-on-restricted-network-enviornment/blob/main/images/02_topolozy.png)



### 4. Gogs 접속

생성된 라우트를 통해 gogs에 접속할 수 있으며 계정 생성 및 서비스 사용이 가능합니다.

![03_gogs](https://github.com/justone0127/Gogs-Deployment-on-restricted-network-enviornment/blob/main/images/03_gogs.png)

