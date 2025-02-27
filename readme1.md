# Cambio de dominio default VPC ROKS

## **Contenido**

1. [Pre-requisitos](#pre-requisitos)  
2. [Ceder gestión del dominio a CIS](#ceder-gestion-del-dominio-a-cis)  
3. [Asignar permisos de gestión de Kubernetes Service sobre CIS](#asignar-permisos-de-gestion-de-kubernetes-service-sobre-cis)  
4. [Crear nuevo dominio en ROKS desde IBM Cloud](#crear-nuevo-dominio-en-roks-desde-ibmcloud)  
5. [Eliminar certificados y configmaps](#eliminar-certificados-y-configmaps)  
6. [Reemplazar certificados](#reemplazar-certificados)  
7. [Reiniciar nodos](#reiniciar-nodos)  
8. [Documentación adicional](#documentacion-adicional)  

---

## **Pre-requisitos** :pencil:

- Instancia de CIS.  
- Clúster ROKS sobre VPC.  
- Dominio propio.  

---

## Ceder gestión del dominio a CIS

1. Agregar dominio.
   </hr>
   ![image](https://github.com/user-attachments/assets/be3b3ed9-2554-4aaa-8405-699a7722249b)  
3. Reemplazar registros NS por los generados por CIS en el gestor del dominio.  
   ![image](https://github.com/user-attachments/assets/85d49608-674b-4333-be65-4e389b42f081)  
   ![image](https://github.com/user-attachments/assets/5efada3c-816c-4e48-82f5-61bf2f77b166)  

## Asignar permisos de gestión de Kubernetes Service sobre CIS

1. Ingresar a la sección de Autorizaciones de IAM en la consola de IBM Cloud.  
   ![image](https://github.com/user-attachments/assets/b4fd4d24-6c6d-4b07-adcf-a460cf5904c4)  
2. Crear autorización entre la instancia de ROKS y CIS.  
   ![image](https://github.com/user-attachments/assets/973f71e9-bb82-4bef-9f4e-78803b0beeb9)  

## Crear nuevo dominio en ROKS desde IBM Cloud

1. Desde la consola de IBM Cloud, en la instancia de ROKS, crear un nuevo default domain asociado al dominio mediante la instancia de CIS.  
   ![image](https://github.com/user-attachments/assets/13ec2237-0154-4c8d-9fde-d17906830cab)  

**Nota**: Al crear el nuevo dominio se generará un nuevo Ingress Controller, un Load Balancer y dos registros DNS en el CIS (un CNAME que direccionará los subdominios al dominio principal y un CNAME para direccionar el tráfico del dominio principal al nuevo Load Balancer).  

## Eliminar certificados y configmaps

1. Iniciar sesión por `oc` CLI.  
2. Identificar certificados y configmaps del Ingress Operator.  
  ```sh
  oc get configmap -n openshift-ingress-operator
  ```
  salida esperada:
  ```
    NAME                       DATA   AGE
    kube-root-ca.crt           1      3h57m
    openshift-service-ca.crt   1      3h57m
    trusted-ca                 1      3h56m
  ```
4. Eliminar los certificados existentes.(kube-root-ca.crt,openshift-service-ca.crt,trusted-ca)
  ```
   oc delete configmap <nombre-configmap> -n openshift-ingress-operator
  ```
## Agregar certificados CA

1. Crear un ConfigMap con el certificado de la CA raíz

```sh
oc create configmap custom-ca \
     --from-file=ca-bundle.crt=</ruta/a/ca-raiz.crt> \
     -n openshift-config
```

2. Crear un secreto con la cadena de certificados y la clave privada.
```sh
oc patch proxy/cluster \
     --type=merge \
     --patch='{"spec":{"trustedCA":{"name":"custom-ca"}}}'
```
3. Crear un secreto con la cadena de certificados y la clave privada
```sh
oc create secret tls <nombre-del-secreto> \
     --cert=</ruta/a/certificado.crt> \
     --key=</ruta/a/clave.key> \
     -n openshift-ingress
```
4. Actualizar la configuración del Ingress Controller
 ```sh
oc patch ingresscontroller.operator default \
     --type=merge -p \
     '{"spec":{"defaultCertificate": {"name": "<nombre-del-secreto>"}}}' \
     -n openshift-ingress-operator
 ```

## Reiniciar nodos

Desde la consola de ibmcloud reiniciar los workeres del cluster.
![image](https://github.com/user-attachments/assets/a959f5c2-de74-4c6e-bc58-63656c322ae3)


## Documentacion adicional

- [Documentacion cambio de cominio con cis](https://cloud.ibm.com/docs/openshift?topic=openshift-ingress-domains&interface=ui)
- [Documentacion cambio de certificado](https://docs.openshift.com/container-platform/4.17/security/certificates/replacing-default-ingress-certificate.html)
