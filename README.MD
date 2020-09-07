
# Crear y actualizar automáticamente **CERTIFICADOS** SSL Seguros con **CERT-MANAGER** y **NGINX-INGRESS-CONTROLLER** **LETSENCRIPT** en **KUBERNETES**

# Introducción

Los **INGRESS** de kubernetes son recursos que permiten dirigir el trafico desde el exterior hacia servicios que corren dentro del cluster.
Un INGRESS dispone de propiedades que podemos modificar a según nuestra necesidad, como por ejemplo reglas de seguridad que nos permiten aceptar o denegar peticiones HTTP/HTTPS, dirigir el tráficos a un servicio u otro según el HOSTNAME de la Request, y lo mas importante es la sección TLS que nos permite asignar un **SECRET SSL**  (certificados SSL .crt y .key guardados como recurso **SECRET** de kubernetes). 

Con **CERT-MANAGER** aprenderemos a crear y actualizar automáticamente **CERTIFICADOS** SSL Seguros con **LETSENCRIPT** en nuestro cluster.

# Que es **CERT-MANAGER**

**CERT-MANAGER** es un controlador nativo que gestiona certificados en Kubernetes. Puede ayudar a emitir certificados de una variedad de fuentes, como Let’s Encrypt, HashiCorp Vault, Venafi, un par de claves de firma simple o autofirmado.

Asegurará que los certificados sean válidos y estén actualizados, e intentará renovar los certificados a la hora configurada antes de su vencimiento.

Se basa libremente en el trabajo de kube-lego y ha tomado prestada cierta sabiduría de otros proyectos similares como kube-cert-manager.
 

# CRDS 

**CERT-MANAGER** Dispone de sus propios CRDS (**Custom Resource Deﬁnitions**, Permiten a los usuarios de Kubernetes extender la funcionalidad base sin modiﬁcar el código de Kubernetes), los cuales tendremos disponibles cuando instalemos cert-manager

los CRDS son los siguientes :

    - Issuer
    - ClusterIssuer
    - Certificate
    - certificaterequests
    - challenges.acme
    - orders

En este post nos centraremos en Issuer, ClusterIssuer, Certificate que son los que usaremos para configurar nuestros ingress.

# Definición
## **Issuer y ClusterIssuers** (Emisor):
Issuers (Emisor) y ClusterIssuers (Emisor a nivel de cluster) parecido a los recursos (Rol, ClusterRol) son recursos de Kubernetes que representan a las autoridades de certificación (CA) que pueden generar certificados firmados cumpliendo con las solicitudes de firma de certificados. Todos los certificados de cert-manager requieren un emisor referenciado que esté listo para intentar cumplir con la solicitud.

## **Certificate**
cert-manager tiene el concepto de Certificados que definen un certificado x509 deseado que se renovará y mantendrá actualizado. Un Certificado es un recurso con espacio de nombres que hace referencia a un Issuer o ClusterIssuer que determina qué cumplirá con la solicitud de certificado.

Cuando se crea un certificado, cert-manager crea un recurso CertificateRequest correspondiente que contiene la solicitud de certificado x509 codificada, la referencia del emisor y otras opciones basadas en la especificación del recurso Certificate como el nombre de dominio y otras mas.

---


# Requisitos previos
Antes de comenzar con esta guía, asegúrese de contar con lo siguiente:

    - Un clúster de Kubernetes 1.10

    - La herramienta de línea de comandos kubectl instalada en su equipo local y configurada para conectarse a su clúster. Puede leer más sobre la instalación de kubectl en la documentación oficial.

    - Tener Instalado la herramienta HELM, ver pagina oficial

    - Un nombre de dominio y registros DNS A que puede orientar al Ingress-controller (balanceador de carga utilizado por el Ingress).

# STEP 1 Instalar NGINX-INGRESS-controller

-  for helm version 2
> **CMD**: helm install stable/nginx-ingress --name nginx-controller

- for helm version 3
> **CMD**: helm install quickstart stable/nginx-ingress
- ver servicios
> **CMD**: watch -x kubectl get svc | grep nginx-ingress

---
    ingresscontroller-nginx-ingress-controller        LoadBalancer   10.28.15.176   34.105.252.88   80:31743/TCP,443:32502/TCP  28d
    ingresscontroller-nginx-ingress-default-backend   ClusterIP      10.28.12.140   <none>          80/TCP                      28d
---

# STEP 2 Asignar Nombre de dominion (HOSTNAME)

Apuntar el nombre de dominio a la ip del loadBalancer/nodePort, asi cada vez que llamemos al nombre de dominio nuestro ingress-controller recibirá la petición y redirigirá el trafico a nuestro ingrees que a su vez pasara el trafico al servicio configurado en el ingress.

# STEP 3 APP DEMO
 Desplegaremos una app DEMO en nuestro cluster de kubernetes.

 Deployment:
---
        apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
        kind: Deployment
        metadata:
            name: nginx-deployment
        spec:
            selector:
                matchLabels:
                    app: nginx
        replicas: 2 # tells deployment to run 2 pods matching the template
        template:
            metadata:
            labels:
                app: nginx
            spec:
                containers:
                - name: nginx
                    image: nginx:1.14.2
                    ports:
                    - containerPort: 80

> **CMD**: kubectl apply -f deployment.yaml
---

Service 
---
    apiVersion: v1
    kind: Service
    metadata:
        labels:
            app: nginx
        name: nginx-service
    spec:
        ports:
            - name: '8080'
            port: 8080
            targetPort: 80
        selector:
            app: nginx
    status:
    loadBalancer: {}

> **CMD**: kubectl apply -f service.yaml
---

Ingress
---
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
        name: ingress
        annotations:
            kubernetes.io/ingress.class: "nginx"
    spec:
        rules:
        - host: hostnameExample.com #Nuestro nombre de dominio configurado en el DNS
            http: &http_rules
            paths:
            - path: /
                backend:
                serviceName: nginx-service
                servicePort: 8080

> **CMD**: kubectl apply -f ingress.yaml
---

Testeamos nuestra app accediendo a la direccion http://www.hostnameExample.com #Nuestro nombre de dominio configurado en el DNS

# STEP 4 Instalamos cert-manager

Instalar con KUBECTL
- Kubernetes 1.16+
> **CMD**: kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.1/cert-manager.yaml

 Kubernetes <1.16
> **CMD**: kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.1/cert-manager-legacy.yaml

**WARNING**: si estas usando GKE (Google Kubernetes Engine) puede que recibas un error de permission denied cuando crees los resources asi que recomiendo ejecutar el siguiente comando.

> **CMD**: kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user=$(gcloud config get-value core/account)

**Note**: Por defecto los resources serán instalados en el namespace cert-manager.

# Instalar con HELM (recomendado)
## **Prerequisite**s :

    - Helm v2 or v3 installed

## **Pasos**:
> **CMD**: kubectl create namespace cert-manager

> **CMD**: helm repo add jetstack https://charts.jetstack.io

> **CMD**: helm repo update



-  Helm v3+

> **CMD**: helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.0.1 --set installCRDs=true

-  Helm v2+
> **CMD**: helm install --name cert-manager --namespace cert-manager --version v1.0.1 --set installCRDs=true jetstack/cert-manager

- verificando la instalacion
> **CMD**: kubectl get pods --namespace cert-manager

**Note**: importante el flag **--set installCRDs=true**


---
        NAME                                       READY   STATUS    RESTARTS   AGE
    cert-manager-5c6866597-zw7kh               1/1     Running   0          2m
    cert-manager-cainjector-577f6d9fd7-tr77l   1/1     Running   0          2m
    cert-manager-webhook-787858fcdb-nlzsq      1/1     Running   0          2m
---

# Crear Recursos cert-manager necesarios

## **Issuer**

**Note**: el siguiente manifesto usara LETSNECRIPT con entorno de **producción**, pero tambien se puede testear con entorno de **staging**.

**NOTE**: ACME significa **Automatic Certificate Management Environment**, Protocolo implementado por Let’s Encrypt. Software compatible con este protocolo que puede usar para comunicarse con Let’s Encrypt a la hora de solicitar un certificado.

**LetsEncript ACME URL PROD** : https://acme-v02.api.letsencrypt.org/directory

**LetsEncript ACME URL STAGING** : https://acme-staging-v02.api.letsencrypt.org/directory

---
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
        name: letsencrypt-prod
    spec:
        acme:
            # El url del servidor ACME
            server: https://acme-v02.api.letsencrypt.org/directory
            # Email usado para el registro ACME
            email: meialExample.provider.com # un email valido
            # Nombre del secret usado para guardar la key privada de la cuenta ACME de letsencript
            privateKeySecretRef:
                name: letsencrypt-prod
            # Activar el HTTP-01 challenge provider
            solvers:
            - http01:
                ingress:
                    class:  nginx
---

Este recurso va a generar un secret SSL automáticamente llamado **letsencrypt-prod**, que contiene la key privada de la cuenta ACME de letsencript, que usara para pedir certificados.

Modificamos el ingress de nuestra app DEMO,
añadiendo en **annotations** la linea 

    cert-manager.io/issuer: "letsencrypt-prod" # referencia a nuestro issuer (emisor) que pedira los certificados a letsencript


y en **spec** :

    tls:
    - hosts:
        - dominioExample.com # Nuestro nombre de dominio configurado en el DNS
        secretName: customsecretname # (debe ser lower case) este nombre lo tomara automáticamente nuestro Issuer (Emisor) para generar el SECRET.


tiene que quedar asi:

Ingress
---
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
        name: ingress
        annotations:
            kubernetes.io/ingress.class: "nginx"
            cert-manager.io/issuer: "letsencrypt-prod" # muy importante esta linea, hace referencia a nuestro emisor el cual pedirá Certificados
    spec:
        tls:
        - hosts:
            - dominioExample.com # Nuestro nombre de dominio configurado en el DNS
            secretName: customsecretname # debe ser lower case, es el nombre de nuestro secret que se generará automaticamente.
        rules:
        - host: hostnameExample.com # Nuestro nombre de dominio configurado en el DNS
            http: &http_rules
            paths:
            - path: /
                backend:
                serviceName: nginx-service
                servicePort: 8080

ejecutamos:
> **CMD**: kubectl apply -f ingress.yaml

vemos nuestro ingress

> **CMD**: kubectl get ing

        NAME                  HOSTS                         ADDRESS        PORTS     AGE
      ingress         admin.server.aelainnova.com        34.89.72.176      80, 443   3h


verificamos que nuestro certificado ha sido creado

> **CMD**: kubectl get certificates

    NAME                   READY   SECRET             AGE
    selfsigned-cert        True   letsencrypt-prod   30m
    selfsigned-cert-test   True   secret-test        54s

verificamos que nuestro secreto ha sido creado

> **CMD**: kubectl get secrets

    NAME                                  TYPE                                  DATA   AGE  
    letsencrypt-prod                      Opaque                                1      20m
    customsecretname-7wtck                Opaque                                1      24s


Si todo ha ido bien probamos ingresar a nuestra URL https://hostnameExample.com, y podremos verificar que el sitio tiene un candado **VERDE**
