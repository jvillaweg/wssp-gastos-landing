# Wssp gastos - Despliegue en AWS

Este proyecto contiene la infraestructura y el sitio web estático para **Wssp gastos**, una aplicación simple que incluye una página principal y una política de privacidad, desplegada en AWS usando S3 + CloudFront.

## Estructura del proyecto

```
├── site/
│   ├── index.html              # Página principal
│   └── privacy-policy.html     # Política de privacidad
├── infra/
│   └── cloudformation.yaml     # Infraestructura como código
└── README.md                   # Este archivo
```

## Características

- **Sitio estático** sin backend ni base de datos
- **Diseño responsive** con soporte para modo oscuro
- **Política de privacidad completa** en español
- **Infraestructura AWS** con S3 + CloudFront + OAC
- **HTTPS obligatorio** y soporte para dominio personalizado
- **Sin JavaScript** - solo HTML y CSS

## Prerrequisitos

- AWS CLI configurado con credenciales apropiadas
- Permisos para crear recursos: S3, CloudFront, IAM
- (Opcional) Dominio personalizado y certificado ACM

## Despliegue

### 1. Crear/actualizar la infraestructura

**Opción A: Sin dominio personalizado**
```bash
aws cloudformation deploy \
  --template-file infra/cloudformation.yaml \
  --stack-name wssp-gastos-site \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides ProjectName=wssp-gastos
```

**Opción B: Con dominio personalizado**
```bash
aws cloudformation deploy \
  --template-file infra/cloudformation.yaml \
  --stack-name wssp-gastos-site \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    ProjectName=wssp-gastos \
    DomainName=tu-dominio.com \
    AcmCertificateArn=arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012
```

> **Nota:** El certificado ACM debe estar en la región `us-east-1` para CloudFront.

### 2. Obtener información del stack

```bash
aws cloudformation describe-stacks \
  --stack-name wssp-gastos-site \
  --query "Stacks[0].Outputs"
```

### 3. Subir archivos al bucket

```bash
# Obtener el nombre del bucket
BUCKET=$(aws cloudformation describe-stacks \
  --stack-name wssp-gastos-site \
  --query "Stacks[0].Outputs[?OutputKey=='BucketName'].OutputValue" \
  --output text)

# Subir archivos
aws s3 sync ./site s3://$BUCKET/ --delete
```

### 4. Obtener la URL del sitio

```bash
# Obtener la URL del sitio
SITE_URL=$(aws cloudformation describe-stacks \
  --stack-name wssp-gastos-site \
  --query "Stacks[0].Outputs[?OutputKey=='SiteURL'].OutputValue" \
  --output text)

echo "Sitio disponible en: $SITE_URL"
```

### 5. (Opcional) Invalidar caché de CloudFront

Si realizas cambios en los archivos, invalida la caché:

```bash
# Obtener el Distribution ID
DISTRIBUTION_ID=$(aws cloudformation describe-stacks \
  --stack-name wssp-gastos-site \
  --query "Stacks[0].Outputs[?OutputKey=='CloudFrontDistributionId'].OutputValue" \
  --output text)

# Invalidar caché
aws cloudfront create-invalidation \
  --distribution-id $DISTRIBUTION_ID \
  --paths "/*"
```

## Configuración de dominio personalizado

Para usar un dominio personalizado:

1. **Registra el dominio** o asegúrate de tener acceso al DNS
2. **Solicita un certificado SSL** en AWS ACM (región `us-east-1`)
3. **Despliega con parámetros** de dominio y certificado
4. **Configura el DNS** apuntando a la distribución de CloudFront:
   ```
   CNAME: tu-dominio.com -> d1234567890123.cloudfront.net
   ```

## Recursos creados en AWS

La plantilla de CloudFormation crea:

- **S3 Bucket** - Almacenamiento de archivos estáticos
- **CloudFront Distribution** - CDN global con HTTPS
- **Origin Access Control (OAC)** - Acceso seguro de CloudFront al bucket
- **Bucket Policy** - Permisos restrictivos para el bucket

## Características de seguridad

- ✅ Bucket S3 **completamente privado** (sin acceso público)
- ✅ Acceso **solo mediante CloudFront** con OAC
- ✅ **HTTPS obligatorio** (redirección automática)
- ✅ **Cifrado en reposo** en S3
- ✅ **Versioning habilitado** en S3

## Costos aproximados

- **S3**: ~$0.023/GB/mes + $0.0004/1000 requests
- **CloudFront**: ~$0.085/GB transferido + $0.0075/10000 requests
- **Route 53** (si usas dominio): ~$0.50/mes por hosted zone

> Para un sitio pequeño: **< $5/mes**

## Desarrollo local

Para probar localmente:

```bash
# Navegar al directorio del sitio
cd site

# Servir con Python (si tienes Python instalado)
python -m http.server 8000

# O usar cualquier servidor estático
# npx serve .
# php -S localhost:8000
```

Visita `http://localhost:8000`

## Limpieza

Para eliminar todos los recursos:

```bash
# Vaciar el bucket primero
aws s3 rm s3://$BUCKET --recursive

# Eliminar el stack
aws cloudformation delete-stack --stack-name wssp-gastos-site
```

## Estructura de URLs

- `/` - Página principal
- `/privacy-policy.html` - Política de privacidad

## Soporte

Para problemas de privacidad o datos: `privacidad@wssp-gastos.example.com`

---

© 2025 Wssp gastos
