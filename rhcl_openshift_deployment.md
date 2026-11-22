# Despliegue de Red Hat Connectivity Link (RHCL) + Service Mesh 3

### **Paso 1: Configurar los Emisores de Certificados (cert-manager)**

1. Ve a **Administration** > **Namespaces** > **Create Namespace**.
2. Crea los siguientes namespaces:
   - `istio-csr` (Agente de CSR, Certificate Signing Request)
   - `istio-system` (Plano de control de Service Mesh)
3. En la consola web de OpenShift, haz clic en el ícono de **"+" (Import YAML)**.
4. Pega el siguiente manifiesto y haz clic en **Create**:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: rhcl-root-ca
  namespace: istio-system
spec:
  isCA: true
  commonName: rhcl-root-ca
  secretName: rhcl-root-ca-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: rhcl-ca-issuer
spec:
  ca:
    secretName: rhcl-root-ca-secret
```

5. Despliega el recurso IstioCSR apuntando al emisor **rhcl-ca-issuer** que acabamos de crear:

```yaml
apiVersion: operator.openshift.io/v1alpha1
kind: IstioCSR
metadata:
  name: default
  namespace: istio-csr
spec:
  istioCSRConfig:
    certManager:
      issuerRef:
        group: cert-manager.io
        kind: ClusterIssuer
        name: rhcl-ca-issuer
    istio:
      namespace: istio-system
      revisions:
        - default
    istiodTLSConfig:
      certificateDuration: 1h0m0s
      certificateRenewBefore: 30m0s
      maxCertificateDuration: 1h0m0s
      privateKeyAlgorithm: ECDSA
      privateKeySize: 256
      trustDomain: cluster.local
    logFormat: text
    logLevel: 1
```

---

### **Paso 2: Crear los Namespaces Base**

1. Ve a **Administration** > **Namespaces** > **Create Namespace**.
2. Crea los siguientes namespaces:
   - `ingress-gateway` (Pasarelas de entrada)
   - `rhcl-system` (Controladores de Red Hat Connectivity Link)
   - `demo-app` (Aplicación de prueba)

---

### **Paso 3: Instalar los Operadores Principales**

1. Ve a **Operators** > **OperatorHub**.
2. Instala **Red Hat OpenShift Service Mesh 3** (en el namespace `openshift-operators`).
3. Busca e instala **Red Hat Connectivity Link** (configurado para gestionar `rhcl-system`).

---

### **Paso 4: Desplegar el Plano de Control de Istio y el Control Global de Kuadrant**

1. Haz clic en **"+" (Import YAML)** y despliega el plano de control:

```yaml
apiVersion: sailoperator.io/v1
kind: Istio
metadata:
  name: default
spec:
  namespace: istio-system
  updateStrategy:
    inactiveRevisionDeletionGracePeriodSeconds: 30
    type: InPlace
  version: v1.31.1
```

2. Inicializa el objeto de control global de Kuadrant en `rhcl-system` (necesario para habilitar el estado `Enforced: True` en las políticas):

```yaml
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant
  namespace: rhcl-system
spec: {}
```

---

### **Paso 5: Desplegar la Aplicación `httpbin` (Adaptada para OpenShift)**

1. Haz clic en **"+" (Import YAML)**.
2. Pega el manifiesto (parcheado a `targetPort: 8080` para evitar bloqueos por ejecutar como usuario _non-root_):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
  namespace: demo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
  template:
    metadata:
      labels:
        app: httpbin
    spec:
      containers:
        - name: httpbin
          image: kennethreitz/httpbin
          command: ["gunicorn", "httpbin:app", "-b", "0.0.0.0:8080"]
          ports:
            - containerPort: 8080
              protocol: TCP
          imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  namespace: demo-app
spec:
  ports:
    - name: http
      port: 80
      targetPort: 8080
  selector:
    app: httpbin
```

---

### **Paso 6: Generar el Certificado TLS para el Ingress**

Crea el secreto de certificado en `ingress-gateway` que utilizará el listener HTTPS del Gateway:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: httpbin-tls-cert
  namespace: ingress-gateway
spec:
  secretName: httpbin-tls-cert
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
  dnsNames:
    - "httpbin.apps.spena-ocp.undani.com"
```

---

### **Paso 7: Crear el Recurso Gateway (HTTP/HTTPS)**

Despliega el `Gateway` con soporte para puerto 80 e inspección/terminación TLS en puerto 443:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public-gateway
  namespace: ingress-gateway
spec:
  gatewayClassName: istio
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: httpbin-tls-cert
      allowedRoutes:
        namespaces:
          from: All
```

---

### **Paso 8: Exponer mediante OpenShift Route (TLS Passthrough)**

Para permitir que el tráfico externo atraviese la capa de OpenShift y llegue directamente a Envoy, crea la siguiente `Route` nativa:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: httpbin-route-ext
  namespace: ingress-gateway
spec:
  host: "httpbin.apps.spena-ocp.undani.com"
  to:
    kind: Service
    name: public-gateway-istio
    weight: 100
  port:
    targetPort: https
  tls:
    termination: passthrough
    insecureEdgePolicy: Redirect
```

---

### **Paso 9: Enrutar la Aplicación con `HTTPRoute`**

Enlaza tu dominio real con la pasarela y la aplicación `httpbin`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: httpbin-route
  namespace: demo-app
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: public-gateway
      namespace: ingress-gateway
  hostnames:
    - "httpbin.apps.spena-ocp.undani.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - group: ""
          kind: Service
          name: httpbin
          port: 80
```

---

### **Paso 10: Aplicar la Política de Rate Limiting (Kuadrant v1 / CEL)**

Aplica la regla de limitación (máximo 5 peticiones GET cada 10 segundos) utilizando la sintaxis formal en expresiones CEL (`predicate`):

```yaml
apiVersion: kuadrant.io/v1
kind: RateLimitPolicy
metadata:
  name: httpbin-ratelimit
  namespace: demo-app
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: httpbin-route
  limits:
    "to-many-requests":
      rates:
        - limit: 5
          window: 10s
      when:
        - predicate: request.method == 'GET'
```

---

### **Paso 11: Validación y Demostración**

Ejecuta una ráfaga desde la terminal PowerShell para comprobar que la regla se aplica en tiempo real:

```powershell
for ($i=1; $i -le 8; $i++) {
    $res = Invoke-WebRequest -Uri "https://httpbin.apps.spena-ocp.undani.com/get" -SkipCertificateCheck -UseBasicParsing
    Write-Output "Peticion $i -> Codigo: $($res.StatusCode)"
}
```

- **Resultado esperado:** Peticiones 1 a 5 responden `200 OK`; peticiones 6 a 8 responden `429 Too Many Requests`.
