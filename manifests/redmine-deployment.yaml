apiVersion: apps/v1
kind: Deployment
metadata:
  name: redmine
  namespace: pmotrack
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redmine
  template:
    metadata:
      labels:
        app: redmine
    spec:
      containers:
        - name: redmine
          image: redmine:5
          ports:
            - containerPort: 3000
          env:
            - name: REDMINE_DB_POSTGRES
              value: postgres            # nombre del Service de PostgreSQL
            - name: REDMINE_DB_DATABASE
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_DB
            - name: REDMINE_DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_USER
            - name: REDMINE_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
          # requests.cpu es OBLIGATORIO para que el HPA pueda calcular el % de uso
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
